# Task

Beware Maniab’s twisted RSA, its bizarre parameters challenge conventional attacks.

nc 65.109.189.191 17777

nc 65.109.194.34 17777


# Writeup (Sage)
```
#!/usr/bin/env sage
from Crypto.Util.number import bytes_to_long, long_to_bytes

nbit = int(1024)

# I think my sage path might be messed up.
sys.path = [''] + sys.path

from maniab_modified import maniab, encrypt

def gen_test_data():
	key = maniab(nbit)
	u, v, N = key
	m = bytes_to_long(b'test flag')
	c = encrypt(m, key)

	return v, N, c

if False:
	#v, N, c = gen_test_data()
	#print(v, N, c)

	# Cached problem instance.
	v, N, c = 40984449453209765166147134085771906481253922367946701120273927215438133227718597520873315286899462656723270309465684401551535854925740862826193174209855162205435127657788925387022137417975526143771677080575554491942279141312263491334931285266110819690282147785876977375325880610435128213773488346221445401927216565978139541342151139727435442811788551094771131646543672820101970683248167505042307774884031982552441787758526959707815882137273520976215247674943190839489127515686205883917373493421209740714228064927253557474653962698954946637251668484551088746508845779831265928569121500582207323405141999864002263915099970489820336892397273662653479986247475429023320793759213555676345385734523730129114214853868371795106712758113395660597338268305256216182661534995847125223285120109125999548536404697200695102503645322966324884700665552112214567942367673474800148473728534842831240136795971650074508176387424115049601666285260828187853129667366573370045798923469560041833491741988778781455453508720317495008541123877608609553050235376894386207331752837783367804029659951117524013203515881441768038773923851271964607485183558704446152825252997008245070667685145180653532468785383781926712383403670827633225881669968284284281363639203, 15333805590662853218544452051320513438074360649643938551670690610467756191973204792814033447963921045504231595632789380433731904067109355098914309186038173250671474747236985673762289332057204446724337575670301335753486855772992012614186868034070109852095919747417752557677152971281025214597451541020615668974060948977947015577608822281768262908436778635462045248639299714907427771791988841581282435751512772040963844850954656966405364114630395737009615554473936215539049897132206939143909015651920004517618373024538179550745151165511865073373916476028065405561126556191097239550384160634758363677127320014251987338403, 6495632329972313849658464146505268679069567959549444207037806280460910939066761971591387053351728000376053362983004479875812654522527621507268859409465773392103271645730552093345627675985948061573993295595948422908773984384179424723389097541582171364279734252530806566139202324660683644065390539139063945707049932906552512338115509461261348600999175597381181836459959863305442630351527270221837420533153699428658147014095979806691118047027058079612244982746112129120794275394763066250977754529884499659999230601727968496672594772155066903654997228835102091886313218378683684959369422251011203348599194982475776852377
else:
	# Get instance from server.
	from pwn import remote, context
	context.log_level = 'debug'
	r = remote('65.109.194.34', 17777)
	r.sendline(b'p')
	r.recvuntil(b'u = REDACTED')

	r.recvuntil(b'v = ')
	v = int(r.recvline())

	r.recvuntil(b'N = ')
	N = int(r.recvline())

	r.sendline(b'e')
	r.recvuntil(b'c = ')
	c = int(r.recvline())

	r.close()

# Find small roots of:
#
# 1 + k * (N + 1 - 2 * x)^2 = 0 mod v
# 1 + k * (N + 1)^2 - 4 * (N + 1) * k * x + 4 * k * x^2 = 0 mod v
#
# where x = avg(p, q)

# Hope that s > 3/4. Should occur at least 1 in 4 tries.
u_max = int(N ^ (2 - sqrt(2 + 3/4)))
x_max = int(((9/8) * N)^(1/2))
k_max = (u_max * v) // (N + 1 - 2 * x_max)^2

print(n(log(u_max, N)))
print(n(log(k_max, N)))
print(n(log(x_max, N)))
print(n(log(v, N)))

max_eqn_power = 3

R.<k,x> = PolynomialRing(ZZ, 2)
eqn = 1 + k * (N + 1 - 2 * x)^2

# Make equation monic
eqn = sum(mon * ((eqn.monomial_coefficient(mon) / 4) % v) for mon in eqn.monomials())

# Build table of monomials:
monomials = []
monomial_info = dict()
det = 1
for i in range(max_eqn_power + 1):
	for k_shift in range(max_eqn_power + 1 - i):
		k_pow = i + k_shift
		for x_shift in range(2):
			x_pow = 2 * i + x_shift

			idx = len(monomials)
			monomial = k^k_pow * x^x_pow
			monomials.append(monomial)
			monomial_info[monomial] = (idx, i, k_shift, x_shift)

			det *= k_max^k_pow * x_max^x_pow / v^i

print(monomials)
print(monomial_info)
print(n(det^(1/len(monomials))))

A = []

# For each monomial, add the basis vector coorsponding to it on the diagonal to the lattice.
for leading_monomial, (_, i, k_shift, x_shift) in monomial_info.items():
	this_eqn = v^(max_eqn_power - i) * eqn^i * k^k_shift * x^x_shift
	#print(this_eqn)

	row = [0] * len(monomials)
	for mon in this_eqn.monomials():
		coeff = this_eqn.monomial_coefficient(mon)
		if mon != leading_monomial:
			coeff = coeff % v^max_eqn_power
		idx = monomial_info[mon][0]
		row[idx] += coeff * mon(k_max, x_max)
	A.append(row)

A = matrix(ZZ, A)
#print(n(abs(A.determinant())**(1/A.nrows()) / v^max_eqn_power))
sol = A.LLL()

I = []
for i in range(5):
	print(n(sum(abs(s) for s in sol[i]) / v^max_eqn_power))
	poly = 0
	for j, mon in enumerate(monomials):
		scale = mon(k_max, x_max)
		assert sol[i][j] % scale == 0
		poly += mon * (sol[i][j] // scale)
	I.append(poly.change_ring(QQ))

I = Ideal(I)
sol_k_x = I.variety()
print(sol_k_x)

for k_x in sol_k_x:
	k = k_x['k']
	x = k_x['x']

	# p * q = N
	# p + q = 2 * x
	# => p and q are the roots of u^2 - 2 * x * u + N

	R1.<u> = QQ['u']
	roots = (u^2 - 2 * x * u + N).roots()
	p = roots[0][0]
	q = roots[1][0]

	phi = (p - 1) * (q - 1)
	e = 1234567891
	d = pow(e, -1, phi)
	m = pow(c, d, N)
	print(long_to_bytes(m))
```
