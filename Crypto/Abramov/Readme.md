# Task

Abramov’s complex numbers are so precise, even MATLAB would demand a PostDoc to verify them!

Note: In the output.txt please change the z ** s to z ** t!

# Writeup

the abramov thing can be solved using pslq 
```
from mpmath import mp, pslq

mp.dps = 1500
yp = mp.mpc("0.6446...", "0.7644...")   # y**p
yq = mp.mpc("0.8793...", "-0.4761...")  # y**q

dp = mp.atan2(yp.imag, yp.real)  # argument of complex number
dq = mp.atan2(yq.imag, yq.real)  

P, Q, M = pslq([dq, -dp, -2*mp.pi], maxcoeff=10**180, maxsteps=100000)
print(P)
print(Q)
```
