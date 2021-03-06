---
path: /posts/2020-04-27-ijctf-2020-writeups
title: IJCTF 2020 Writeups
date: 2020-04-27
tags: ctf,infosec,writeup,crypto
---

I played IJCTF this weekend on team `misc` and we came 2nd. I was only capable of solving some of the easier crypto challenges but they were a good set of challenges and I'm very pleased with them.

- crypto
    - [Klepto](#klepto)
    - [Nibiru](#nibiru)


---

# Klepto <a name="klepto"></a>

### crypto (932pts)

> My friend found a more secure way to do RSA and he insisted that I use it. I have a feeling something is weird though, he really really wanted me to use it.
> 
> Author: Harsh

`number.py`:

```python
#!/usr/bin/env python3

from random import getrandbits, randrange

def generate():
	IV = 5326453656607417485924926839199132413931736025515611536784196637666048763966950827721156230379191350953055349475955277628710716247141676818146987326676993279104387449130750616051134331541820233198166403132059774303658296195227464112071213601114885150668492425205790070658813071773332779327555516353982732641; seed = 0; temp = [0, 0]; key = 0
	while(key != 2):
		if key == 0:
			seed = getrandbits(1024) | (2 ** 1023 + 1)
		seed_ = seed ^ IV; n = seed_ << 1024 | getrandbits(1024); seed = n//seed | 1
		while(1):
			seed += 2; pi = seed - 1; b = 0; m = pi;
			while (m & 1) == 0:
				b += 1
				m >>= 1
			garbage = []; false_positive = 1
			for i in range(min(10, seed - 2)):
				a = randrange(2, seed)
				while a in garbage:
					a = randrange(2, seed)
				garbage.append(a); z = pow(a, m, seed)
				if z == 1 or z == pi:
					continue
				for r in range(b):
					z = (z * z) % seed;
					if z == 1:
						break
					elif z == pi:
						false_positive = 0; break
				if false_positive:
					break
			if not false_positive:
				break
		temp[key] = seed; key += 1
	return(temp[0], temp[1])

def egcd(a, b):
	if a == 0:
		return (b, 0, 1)
	else:
		g, y, x = egcd(b % a, a)
		return (g, x - (b // a) * y, y)

def inverse(a, m):
	g, x, y = egcd(a, m)
	if g != 1:
		raise Exception('modular inverse does not exist')
	else:
		return x % m

def RSA():
	(p, q) = (0, 0)
	while(p == q):
		(p, q) = generate()
	n = p * q
	e = 0x10001
	d = inverse(e, (p - 1) * (q - 1))

	return (n, e, d)
```

`enc`:
```
n:  31492593980972292127624243962770751972791586997559415119664067126352874455354554396825959492520866585425367347800693358768073355393830644370233119742738920691048174984688259647179867604016694194959178874017481204930806636137689428300906890690499460758884820784037362414109026624345965004388036791265355660637942956892135275111144499770662995342441666665963192321379766221844532877240853120099814348371659810248827790798567539193378235283620829716728221866131016495050641120781696656274043778657558111564011476899021178414456560902915167861941065608670797745473745460370154604158882840935421329890924498705681322154941
e:  65537
c:  31105943135542175872131854877903463541878591074483146885600982339634188894256348597314413875362761608401326213641601058666720123544299104566877513595233611507482373096711246358592143276357334231127437090663716106190589907211171164566820101003469295773837109380815526746746678580390779170877213166506863176846508933485178812858166031517243792487958635017917996626849408595921536238471169975280762677305759764602707285224588771643832335444552739959904673158661424651074772864245589406308229908379716500604590198490474257870603210120239125184805345188782082520055749851184516545898673495570079185198108523819932428027921
```

## Solution

Adam solved this challenge during the CTF but I decided to do a writeup for it anyway. I used his idea.

The RSA primes are generated by the `generate` function so we begin by analysing the `generate` function. At first glance it seems quite complex, but the entirety of the `while(1)` loop is primality testing. After each round (of which there are two), the value of `seed` is outputted as the prime. The part preceding the `while(1)` loop generates a non neccesarily prime number, so the output is the next prime after that number generated. During the second round, the value of `seed` is equal to the value of `p` (the first generated RSA prime). Hence, we can rewrite the equations:

```
seed_ = p ^ IV;

n = seed_ << 1024 | getrandbits(1024)
=> (p ^ IV) << 1024 + g     , where g is 1024 bits of garbage

seed = n//seed | 1
=> q ~= ((p ^ IV) << 1024 + g)//p
```

It then follows that `p*q = N ~= (p ^ IV) << 1024 + g` and so `N >> 1024 ~= p ^ IV`. We can use this to factorise `N` by bruteforcing values of `p` close to `(N >> 1024) ^ IV`. Once we have the value `p`, we can easily find `q` by computing `N//p`. From there we decrypt the ciphertext with textbook RSA decryption.

Solve script:

```python
from Crypto.Util.number import *

# N >> 1024 ~= p ^ IV
from values import N,e,c,IV

s = (N >> 1024) ^ IV
for t in range(-10000, 10000):
    p = s + t
    q = N//p
    if isPrime(p) and isPrime(q) and p*q == N:
        m = pow(c, inverse(e, (p-1)*(q-1)), N)
        print(long_to_bytes(m).decode())
        exit()
```

Flag: `IJCTF{kl3pt0_n0th1n6_uP_mY_s133v3_numb3r5}`

---

# Nibiru <a name="nibiru"></a>

### crypto (955pts)

> The government has finally declassified Report 00165789 ...
> 
> Author: Harsh

[Nibiru.pdf](./assets/Nibiru.pdf)

## Solution

We are given a pdf of some "government report" that contains a ciphertext. The cipher is described towards the bottom of the pdf. The description explains it pretty well, except it starts indexing at `1`. Since we'll be wanting to work in the ring $\mathbb{Z/26Z}$, we'll need to shift everything by 1. For clarity, here's how the (very slightly modified) encryption and decryption works for a plaintext message $m$ and its ciphertext $c$.

#### Key

Let $K$ be the key. $K$ is a permutation of the alphabet.

Let $0 \leq s \leq 25$ be the initial shift amount.

We denote the index of a letter $l$ in the key with $K(l)$. For example, if the key is the alphabet reversed, $K(Z) = 0$ and $K(A) = 25$.

We denote the letter at index $i$ in the key with $K_i$. For example, if the key is the alphabet reversed, $K_0 = Z$ and $K_{25} = A$.

Note that with this notation, $K_{K(l)} = l$ and $K(K_i) = i$.

#### Encryption

$$
e_i \equiv \begin{cases} K(m_i) + s \pmod{26}, &\quad i = 0 \\ K(m_i) + K(m_{i-1}) + 1 \pmod{26}, &\quad i > 0 \end{cases}
$$

$$
c_i = K_{e_i}, \quad i \geq 0
$$


#### Decryption


$$
d_i \equiv \begin{cases} K(c_i) - s \pmod{26}, &\quad i = 0 \\ K(c_i) - K(m_{i-1}) - 1 \pmod{26}, &\quad i > 0 \end{cases}
$$

alternatively using the identity $K(K_i) = i$, we can write,

$$
d_i \equiv \begin{cases} e_i - s \pmod{26}, &\quad i = 0 \\ e_i - K(m_{i-1}) - 1 \pmod{26}, &\quad i > 0 \end{cases}
$$

$$
m_i = K_{d_i}, \quad i \geq 0
$$

#### Proof of correctness:

We aim to prove $D(E(m)) = m$. We first prove the case for the first character:

$$
\begin{aligned} e_0 &\equiv K(m_0) + s \pmod{26} \\ \implies d_0 &\equiv e_0 - s \pmod{26} \\ &\equiv K(m_0) + s - s \pmod{26} \\ &\equiv K(m_0) \pmod{26} \end{aligned}
$$

For the more general case:

$$
\begin{aligned} e_i &\equiv K(m_i) + K(m_{i-1}) + 1 \pmod{26} \\ \implies d_i &\equiv e_i - K(m_{i-1}) - 1 \pmod{26} \\ &\equiv K(m_i) + K(m_{i-1}) + 1 - K(m_{i-1}) - 1 \pmod{26} \\ &\equiv K(m_i) \pmod{26} \end{aligned}
$$

It follows that $m_i = K_{d_i} = K_{K(m_i)} = m_i$ in both cases.

### Cipher Implementation

The following code implements the cipher:

```python
def encrypt(plaintext, initial_shift, key):
    ciphertext = ''
    shift = initial_shift
    for m_i in plaintext:
        # maintain punctuation, etc.
        if m_i not in key:
            ciphertext += m_i
            continue
        c_i = key[(key.index(m_i) + shift + 1) % 26]
        ciphertext += c_i
        shift = key.index(m_i)
    return ciphertext

def decrypt(ciphertext, initial_shift, key):
    plaintext = ''
    shift = initial_shift
    for c_i in ciphertext:
        if c_i not in key:
            plaintext += c_i
            continue
        m_i = key[(key.index(c_i) - shift - 1) % 26]
        plaintext += m_i
        shift = key.index(m_i)
    return plaintext
```

We can test it works with the given plaintext/ciphertext sample in the pdf:

```python
ciphertext = 'KRSCQ OSHMG VXRRS VUV YSQX, FWADWJFXXNX DLHSSUZGPX LPNV.'
plaintext = 'LOREM IPSUM DOLOR SIT AMET, CONSECTETUR ADIPISCING ELIT.'
key = 'FIREABCDGHJKLMNOPQSTUVWXYZ'
s = 24
assert encrypt(plaintext, s, key) == ciphertext
assert decrypt(ciphertext, s, key) == plaintext
```

### Analysing The Challenge

We are given the ciphertext for some message, but we don't have a key or a shift amount, so we can't decrypt it immediately. However, we are given the first sentence in plaintext, so we can use that to set up constraints on the key.

Consider the variables $A, B, C, \ldots, Z$ as elements in $\mathbb{Z/26Z}$ representing their respective letter's index in the key. For example, suppose the key is the alphabet reversed. Then $Z = 0$ and $A = 25$. It follows that $A \neq B \neq \cdots \neq Z$.

We are trying to solve for 26 unknowns, and we're given 32 letters of plaintext. From the 32 letters of plaintext, we can set up 31 equations (we ignore the first letter which uses the shift as we can just bruteforce the shift later). We form each equation using the decryption formula described above. For example, since the first two letters of plaintext are `"IF"` and the first two letters of the ciphertext are `"TJ"`, we can write the two equations:

$$
\begin{aligned} T - s &\equiv I \pmod{26} \\ J - I - 1 &\equiv F \pmod{26} \end{aligned}
$$

where $s$ is the initial shift amount. We rewrite the equations as:

$$
\begin{aligned} T - s - I &\equiv 0 \pmod{26} \\ J - I - F - 1 &\equiv 0 \pmod{26} \end{aligned}
$$

We continue this process to get 31 equations.

To represent the equations, we use a 31x27 matrix with each row representing a single equation. Each row is a vector of size 27 and contains only elements in $\{-2, -1, 0, 1\}$. The first element of each row vector represents the coefficient of $A$, the second represents the coefficient of $B$ and so on, up to the 26th element which represents the coefficient of $Z$. The 27th element of each row vector is always `1` and represents the shift by one in each equation.

As an example, the equation $J - I - F - 1 = 0$ is represented by the row vector of size 27 (the top row containing letters is there for illustration purposes):

$$
\mathbf{e_0} = \begin{bmatrix} A &\ldots &F &G &H &I &J &K  &\ldots  &Z &1\\ 0 &\ldots &-1 &0 &0 &-1 &1 &0 &\ldots &0 &1 \end{bmatrix}
$$

With 31 of these vectors, we can set up the matrix equation:

$$
\begin{bmatrix} \mathbf{e_0} \\ \mathbf{e_1} \\ \vdots \\ \mathbf{e_{30}} \end{bmatrix}\begin{bmatrix} A \\ B \\ \vdots \\Z \\ g \end{bmatrix} \equiv \mathbf{0} \pmod{26}
$$

where $\mathbf{e_i}$ is the row vector representing the $ith$ equation and $g$ is some value we are not particularly concerned about.

Our problem now reduces down to solving a system of linear congruences modulo 26. We see that the variables vector $(A, B, \ldots, Z, g)$ is any vector in the kernel of the equations matrix. We can solve this by using the congruences

$$
\begin{bmatrix} \mathbf{e_0} \\ \mathbf{e_1} \\ \vdots \\ \mathbf{e_{30}} \end{bmatrix}\begin{bmatrix} A \\ B \\ \vdots \\Z \\ g \end{bmatrix} \equiv \mathbf{0} \pmod{2}\quad \text{and} \quad \begin{bmatrix} \mathbf{e_0} \\ \mathbf{e_1} \\ \vdots \\ \mathbf{e_{30}} \end{bmatrix}\begin{bmatrix} A \\ B \\ \vdots \\Z \\ g \end{bmatrix} \equiv \mathbf{0} \pmod{13}
$$

We take the basis vectors of the kernel of the equations matrix in $\mathbb{F}_2$ and the basis vectors of the kernel of the equations matrix in $\mathbb{F}_{13}$ and combine their results to get solutions in $\mathbb{Z/26Z}$. The nullity of the matrices in each ring is 2, so there are $26^4$ different possible linear combinations to try (although we could probably reduce this number with a bit more work). We write

$$
\mathbf{v} = c_0\mathbf{b_0} + c_1\mathbf{b_1} + c_2\mathbf{b_2} + c_3\mathbf{b_3}
$$

where $c_0, c_1, c_2, c_3 \in [0, 25]$ and $\mathbf{b_i}$ are the basis vectors. The elements of $\mathbf{v}$ are in $\mathbb{Z/26Z}$. If we compute $\mathbf{v}$ and find that $\mathbf{Ev} = 0$ (where $\mathbf{E}$ is the equations matrix in $\mathbb{Z/26Z}$), then we can add $\mathbf{v}$ as a possible key since it satisfies all the constraints of the system.

Once we have a list of possible keys, we can bruteforce the shift amount for each possible key and see which decrypt the ciphertext properly.

### Implementation

```python
from string import ascii_uppercase
from itertools import product

ciphertext = 'T JABD QL EHZRZBG DPE GKYX HWCROH EM VOZ. ORUVC QRBHAE HP EJWZWYW XJY LYAT RDRA AXCBVMYC, YY YYCC BGUJN MI HUV LVLW ICJP KLTJ RZV ZHNWCXCCH WP RQ PXXP WKCBWVC UUMIE: B VVDIVDZKJAM HP UAJ SKYCUL ZIAQLWO, MJWZNH FI IAJ DHMX JDRA MQOHQ ASBDPC HP JXEOPGU XYOGW. B NVYWF WDQGHR! GAJ OOP QQV P DYCC QKXCXWNMP KDRA K IHUAMC VI PJYV FASA FKCBW MXCBVMYC. BQV IRUVC WUXXCDO FRBHAE HP TUBDINSYW, HJYV PR PY HPP. RCKW PY HPPVH XCBOWIE YZV LAKW MAJ RPHRRS SAKW FRYNZNH PYGAPH VAKW ZIAQLWO CVDIVDZKJAM EKI KDRP GUBDINSYW HPTYM VV OYNZ. WY BJY XASMWL UPT UDLYC DQV RYBODA, CY JKI MPAYVKXCCH I WVCGMNQN RXCW. WR JUCFFH EKDB BJY BABDW, ZJMCCYW AM LI VVREXR AYTGMNQN BHZL. TJYQOZK, HY QQ QHZ IP GHUISCDWRZB HI HLID MCKW PY JKI KDGVRO, K AUV MAJ UESF TVEYTC QB HAJ OXAKAZB UVSVL, RY Q HVKX YY PQPOH IRUVC FCKW PY YLJGT ZHCB THMYW EAJCC: GIAQLWO SVKX. YY DZYMKH FAJ RXCW DXCBVMYC HHRS, XHRS KIVHCKKTJIV. P OVBUGBONH PRL CAVHWRZB QGUSDJ FAJ UESF TQV IDXUVGZK OUTIVGZVH PR YR EQ IIUIDHQNGQN GOFZJC. M NHTJAPA MAJ UESF ZJCC, VI IAJ VID DP UASA DRJIHWLY, HI HLWE GSPMHP OPTCXXC. M KPUVWHH FAJ AMOJ BDRA AAJ XCQIPPCC GOFZJC, BUEASG TOWZ YAJPNRPCOU UHKKBE QE RWFFPE GORELBSJMJC. GBP QP UAJ VPYNHX RTLBCYONPH CMFQ MMUH FI WHGVWH VASA DOUKVJI YZV MMUH.'
known_plaintext = 'I FEAR MY ACTIONS MAY HAVE DOOMED US ALL.'

# CIPHER IMPLEMENTATION
def decrypt(ciphertext, initial_shift, key):
    plaintext = ''
    shift = initial_shift
    for c_i in ciphertext:
        if c_i not in key:
            plaintext += c_i
            continue
        m_i = key[(key.index(c_i) - shift - 1) % 26]
        plaintext += m_i
        shift = key.index(m_i)
    return plaintext

# TRANSFORM A VECTOR (N=26) INTO A KEY
def vec_to_key(vec):
    key = ['?'] * 26
    for i, v in enumerate(vec):
        key[v] = ascii_uppercase[i]
    return ''.join(key)

# SETTING UP SYSTEM OF EQUATIONS
def clean(s):
    return ''.join(c for c in s if c in ascii_uppercase)

# Matrix A with the last entry in each row being 1 (for the shift of 1)
# The first entry is the coefficient for A,
# The second entry is the coefficient for B,
# and so on...
def construct_equations(ciphertext, known_plaintext):
    ciphertext = clean(ciphertext)
    known_plaintext = clean(known_plaintext)
    left = list(ciphertext)[1:]
    shift = list(known_plaintext)
    equals = list(known_plaintext)[1:]
    A = []
    # l - s - e - 1 = 0
    for l, s, e in zip(left, shift, equals):
        eqn = [0]*26 + [1]
        eqn[ascii_uppercase.index(l)] += 1
        eqn[(ascii_uppercase).index(s)] -= 1
        eqn[ascii_uppercase.index(e)] -= 1
        A.append(eqn)
    return A

F = IntegerModRing(26)
EQUATIONS = construct_equations(ciphertext, known_plaintext)
E = Matrix(F, EQUATIONS)

# Bruteforce possible keys!
possible_keys = set()
A = Matrix(GF(2), EQUATIONS)
B = Matrix(GF(13), EQUATIONS)
kA = Matrix(F, A.right_kernel().basis_matrix())
kB = Matrix(F, B.right_kernel().basis_matrix())
basis_vectors = Matrix(F, [*kA, *kB])
for lin_comb in product(range(26), repeat=basis_vectors.nrows()):
    s = basis_vectors.linear_combination_of_rows(lin_comb)
    if (E*s).is_zero():
        key = vec_to_key(s[:26])
        possible_keys.add(key)
print('found', len(possible_keys), 'possible keys')

# Bruteforce the shift for each key
for key in possible_keys:
    for shift in range(1, 26):
        m = decrypt(ciphertext, shift, key)
        if known_plaintext in m:
            print(key, shift)
```

### Capturing The Flag

After decrypting the ciphertext from the pdf, we get the message:

> I FEAR MY ACTIONS MAY HAVE DOOMED US ALL. AFTER MONTHS OF FILLING OUR HOLD WITH TREASURE, WE WERE ABOUT TO SET SAIL WHEN WORD WAS DELIVERED OF AN EVEN GREATER PRIZE: A SARCOPHAGUS OF THE PUREST CRYSTAL, FILLED TO THE BRIM WITH BLACK PEARLS OF IMMENSE VALUE. A KINGS RANSOM! THE MEN AND I WERE OVERTAKEN WITH A DESIRE TO FIND THIS GREAT TREASURE. AND AFTER SEVERAL MONTHS OF SEARCHING, FIND IT WE DID. WHAT WE DIDNT REALIZE WAS THAT THE ENTITY THAT DWELLED INSIDE THAT CRYSTAL SARCOPHAGUS HAD BEEN SEARCHING FORUS AS WELL. IN OUR THIRST FOR POWER AND WEALTH, WE HAD DISCOVERED A TERRIBLE EVIL. IT PREYED UPON OUR FEARS, DRIVING US TO COMMIT HORRIBLE ACTS. FINALLY, IN AN ACT OF DESPERATION TO STOP WHAT WE HAD BECOME, I SET THE SHIP ASHORE ON THE MISSION COAST, IN A COVE WE NAMED AFTER WHAT WE WOULD SOON BRING THERE: CRYSTAL COVE. WE BURIED THE EVIL TREASURE DEEP, DEEP UNDERGROUND. I CONCEALED ITS LOCATION ABOARD THE SHIP AND ARTFULLY PROTECTED IT BY AN UNCRACKABLE CIPHER. I BROUGHT THE SHIP HERE, TO THE TOP OF THIS MOUNTAIN, TO STAY HIDDEN FOREVER. I ENCODED THE FLAG WITH THE VIGENERE CIPHER, FTGUXI ICPH OLXSVGHWSE SOVONL BW DOJOFF DHUDCYTPWMQ. ONE OF THE TWELVE EQUIVALENT KEYS USED TO DECODE THIS MESSAGE WAS USED.'
 
The implementation script above prints all possible keys (of which there are 12):

```python
>>> sage solve.sage
found 676 possible keys
BINTYRHMSXAGLQWEDKPVFCJOUZ 1
WTPMJGBEYVSOLIDRFXUQNKHCAZ 13
DMUEHOWRJQYCLTFGNVAIPXBKSZ 19
PGYODXNCWMBVLRUKATJESIFQHZ 21
NRSGWKFOBTHXLEPCUIYMAQDVJZ 17
SKBXPIAVNGFTLCYQJRWOHEUMDZ 5
JVDQAMYIUCPELXHTBOFKWGSRNZ 7
HQFISEJTAKURLVBMWCNXDOYGPZ 3
UOJCFVPKDEWQLGAXSMHRYTNIBZ 23
ACHKNQUXFRDILOSVYEBGJMPTWZ 11
YXWVUTSQPONMLKJIHGDCBRAEFZ 15
FEARBCDGHIJKLMNOPQSTUVWXYZ 9
```

The plaintext message says that the ciphertext `FTGUXI ICPH OLXSVGHWSE SOVONL BW DOJOFF DHUDCYTPWMQ` has been encrypted with a vigenere cipher using one of the twelve keys found. After attempting each key to decrypt the flag, we find that the key `NRSGWKFOBTHXLEPCUIYMAQDVJZ` gives something that looks like the flag.

Flag: `IJCTF{SCOOBY DOOO HOMOGENOUS SYSTEM OF LINEAR CONGRUENCES}`
