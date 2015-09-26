
##Calculate quantiles along axis while ignoring NaN

We are calculating quantiles along the time axis for time-series classification. The datasets can contain NaNs on non-valid observations (i.e. cloud mask). Numpys np.nanpercentile is very slow due to it looping over the array in pure Python.

This spurred the idea to calculate the quantiles with a new function.

Relevant StackOverflow posts:

 - [Pure Python implementation of percentile](http://stackoverflow.com/questions/2374640/how-do-i-calculate-percentiles-with-python-numpy)

 - [Index 3D array with z-index stored in 2D array](http://stackoverflow.com/a/32091712/4169585)

Semi-relevant:
np.choose() only allows indexing up to 32 choices, i.e. layers in the time-series.


    import numpy as np

Test data.


    # create array of shape(5,100,100) - image of size 10x10 with 5 layers
    test_arr = np.random.randint(0, 10000, 50000).reshape(5,100,100).astype(np.float32)
    np.random.shuffle(test_arr)
    # place random NaN
    rand_NaN = np.random.randint(0, 50000, 500).astype(np.float32)
    for r in rand_NaN:
        test_arr[test_arr == r] = np.NaN

Benchmark numpys nanpercentile function to see what we are working with.


    %timeit np.nanpercentile(test_arr, q=25, axis=0)

    100 loops, best of 3: 5.92 ms per loop
    

The general idea is to
 - find number of valid observations
 - replace NaN with maximum value of array
 - sort values along axis
 - find position of quantile regarding number of valid observations
 - linear interpolation if the position is not int


    # find the number of valid observations over time per location
    valid_obs = np.sum(np.isfinite(test_arr), axis=0)


    # find the maximum of the array and replace NaNs with it
    # this way they will be sorted to the end
    max_val = np.nanmax(test_arr)
    test_arr[np.isnan(test_arr)] = max_val


    # sort array in order to find position of quantiles and push non valid observations to the end
    test_arr = np.sort(test_arr, axis=0)


    # position of desired quantile
    quant = 25
    k_arr = (valid_obs - 1) * (quant / 100.0)


    # floor and ceiling of desired position
    f_arr = np.floor(k_arr).astype(np.int16)
    c_arr = np.ceil(k_arr).astype(np.int16)
    
    # mask where floor, ceiling and desired position are identical
    fc_equal_k_mask = f_arr == c_arr


    # linear interpolation like in numpy percentile
    # takes the value at floor and ceiling position
    # the fractional part of the position is used for interpolation and calculation of quantile value
    floor_val = f_arr.choose(test_arr) * (c_arr - k_arr)
    ceil_val = c_arr.choose(test_arr) + (k_arr - f_arr)
    
    # ATTENTION: if choose() gets more than 32 choices, i.e. layers to choose from, it throws a value error
    # work around this with the helper function based on np.take()
    floor_val = _zvalue_from_index(test_arr, f_arr)
    ceil_val = _zvalue_from_index(test_arr, c_arr)
    
    quant_arr = floor_val + ceil_val
    
    #account for floor == ceiling == desired position since these become 0 in the calculation
    quant_arr[fc_equal_k_mask] = f_arr.choose(test_arr)[fc_equal_k_mask]

All wrapped into a function that takes a 3D ndarray containing NaNs and a list of quantiles to calculate.


    def _zvalue_from_index(arr, ind):
        """private helper function to work around the limitation of np.choose() by employing np.take()
        arr has to be a 3D array
        ind has to be a 2D array containing values for z-indicies to take from arr
        See: http://stackoverflow.com/a/32091712/4169585
        This is faster and more memory efficient than using the ogrid based solution with fancy indexing.
        """
        # get number of columns and rows
        _,nC,nR = arr.shape
        
        # get linear indices and extract elements with np.take()
        idx = nC*nR*ind + nR*np.arange(nR)[:,None] + np.arange(nC)
        return np.take(arr, idx)


    def nan_percentile(arr, q):
        # valid (non NaN) observations along the first axis
        valid_obs = np.sum(np.isfinite(arr), axis=0)
        # replace NaN with maximum
        max_val = np.nanmax(arr)
        arr[np.isnan(arr)] = max_val
        # sort - former NaNs will move to the end
        arr = np.sort(arr, axis=0)
        
        # loop over requested quantiles
        if type(q) is list:
            qs = []
            qs.extend(q)
        else:
            qs = [q]
        if len(qs) < 2:
            quant_arr = np.zeros(shape=(arr.shape[1], arr.shape[2]))
        else:
            quant_arr = np.zeros(shape=(len(qs), arr.shape[1], arr.shape[2]))
        
        result = []
        for i in range(len(qs)):
            quant = qs[i]
            # desired position as well as floor and ceiling of it
            k_arr = (valid_obs - 1) * (quant / 100.0)
            f_arr = np.floor(k_arr).astype(np.int32)
            c_arr = np.ceil(k_arr).astype(np.int32)
            fc_equal_k_mask = f_arr == c_arr
            
            # linear interpolation (like numpy percentile) takes the fractional part of desired position
            floor_val = _zvalue_from_index(arr=arr, ind=f_arr) * (c_arr - k_arr)
            ceil_val = _zvalue_from_index(arr=arr, ind=c_arr) * (k_arr - f_arr)
            
            quant_arr = floor_val + ceil_val
            quant_arr[fc_equal_k_mask] = _zvalue_from_index(arr=arr, ind=k_arr.astype(np.int32))[fc_equal_k_mask]  # if floor == ceiling take floor value
            
            result.append(quant_arr)
        
        return result

Test if the new function returns the same values as np.nanpercentile


    input_arr = np.array(test_arr, copy=True)
    old_func = np.nanpercentile(input_arr, q=range(0,100), axis=0)
    new_func = nan_percentile(input_arr, q=range(0,100))


    for i in range(len(new_func)):
        print i
        print np.allclose(new_func[i], old_func[i,:,:])
        print "--"

    0
    True
    --
    1
    True
    --
    2
    True
    --
    3
    True
    --
    4
    True
    --
    5
    True
    --
    6
    True
    --
    7
    True
    --
    8
    True
    --
    9
    True
    --
    10
    True
    --
    11
    True
    --
    12
    True
    --
    13
    True
    --
    14
    True
    --
    15
    True
    --
    16
    True
    --
    17
    True
    --
    18
    True
    --
    19
    True
    --
    20
    True
    --
    21
    True
    --
    22
    True
    --
    23
    True
    --
    24
    True
    --
    25
    True
    --
    26
    True
    --
    27
    True
    --
    28
    True
    --
    29
    True
    --
    30
    True
    --
    31
    True
    --
    32
    True
    --
    33
    True
    --
    34
    True
    --
    35
    True
    --
    36
    True
    --
    37
    True
    --
    38
    True
    --
    39
    True
    --
    40
    True
    --
    41
    True
    --
    42
    True
    --
    43
    True
    --
    44
    True
    --
    45
    True
    --
    46
    True
    --
    47
    True
    --
    48
    True
    --
    49
    True
    --
    50
    True
    --
    51
    True
    --
    52
    True
    --
    53
    True
    --
    54
    True
    --
    55
    True
    --
    56
    True
    --
    57
    True
    --
    58
    True
    --
    59
    True
    --
    60
    True
    --
    61
    True
    --
    62
    True
    --
    63
    True
    --
    64
    True
    --
    65
    True
    --
    66
    True
    --
    67
    True
    --
    68
    True
    --
    69
    True
    --
    70
    True
    --
    71
    True
    --
    72
    True
    --
    73
    True
    --
    74
    True
    --
    75
    True
    --
    76
    True
    --
    77
    True
    --
    78
    True
    --
    79
    True
    --
    80
    True
    --
    81
    True
    --
    82
    True
    --
    83
    True
    --
    84
    True
    --
    85
    True
    --
    86
    True
    --
    87
    True
    --
    88
    True
    --
    89
    True
    --
    90
    True
    --
    91
    True
    --
    92
    True
    --
    93
    True
    --
    94
    True
    --
    95
    True
    --
    96
    True
    --
    97
    True
    --
    98
    True
    --
    99
    True
    --
    

Speed comparison to see if we are faster than before:


    input_arr = np.array(test_arr, copy=True)
    %timeit np.nanpercentile(input_arr, q=[10,25,50,75,90], axis=0)

    1 loops, best of 3: 603 ms per loop
    


    input_arr = np.array(test_arr, copy=True)
    %timeit nan_percentile(test_arr, q=[10,25,50,75,90])

    100 loops, best of 3: 3.81 ms per loop
    

###RESULT!
We sped up np.nanpercentile by a factor of ~200!
