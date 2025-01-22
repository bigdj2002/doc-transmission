import sys
import os
import tempfile
import json
import numpy as np

from scipy.interpolate import pchip_interpolate

def bdbr_calculation(R1, Q1, R2, Q2, piecewise=0):
    if len(R1) == 0 or len(Q1) == 0 or len(R2) == 0 or len(Q2) == 0:
        raise ValueError("Input vectors must not be empty")
    
    lR1 = np.log(R1)
    lR2 = np.log(R2)

    # rate method
    p1 = np.polyfit(Q1, lR1, 3)
    p2 = np.polyfit(Q2, lR2, 3)

    # integration interval
    min_int = max(min(Q1), min(Q2))
    max_int = min(max(Q1), max(Q2))

    # find integral
    if piecewise == 0:
        p_int1 = np.polyint(p1)
        p_int2 = np.polyint(p2)

        int1 = np.polyval(p_int1, max_int) - np.polyval(p_int1, min_int)
        int2 = np.polyval(p_int2, max_int) - np.polyval(p_int2, min_int)
    else:
        lin = np.linspace(min_int, max_int, num=100, retstep=True)
        interval = lin[1]
        samples = lin[0]
        v1 = pchip_interpolate(np.sort(Q1), np.sort(lR1), samples)
        v2 = pchip_interpolate(np.sort(Q2), np.sort(lR2), samples)
        # Calculate the integral using the trapezoid method on the samples.
        int1 = np.trapz(v1, dx=interval)
        int2 = np.trapz(v2, dx=interval)

    # find avg diff
    avg_exp_diff = (int2 - int1) / (max_int - min_int)
    avg_diff = (np.exp(avg_exp_diff) - 1) * 100
    
    return avg_diff
