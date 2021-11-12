# Implementing DES In Python
In this project I write a basic implementation of the DES Block Cipher. It will encrypt and decrypt 64 bit blocks (in binary or hex) using 64 bit keys (Technically only 56 bits are needed, but the specification includes parity bits that are meant to be dropped, so I stuck with the 64 bit keys.

---
### Overview of The Encryption    
Despite the name DES is not longer the Data Encryption Standard as it was replaced by AES, but it is still an interesting Block Cipher to implement.  
![Fesitel Network Diagram](/images/FIPS_.png)

#### DES Encryption Function
This is the main function that outlines the overall structure of the DES Feistel Network. 
```python
def encrypt(message, key):

    message, key = check_data_and_key(message, key)
    key_schedule = generate_keys(key)

    print('Data Being Encrypted: ', bin_to_hex(message, 64))
    perm = IP(message)
    print('After Initial Permutation: ', bin_to_hex(perm, 64))
    left_half, right_half = perm[0:32], perm[32:64]
    
    print(' ' * 10, 'Left Half  ', 'Right Half ', 'Sub Key  ')
    for loop in range(16):
        left_half, right_half = round(left_half, right_half, key_schedule[loop])
        if loop == 15: # For Some Reason The Algorithm Doesn't Swap The Last Two Keys So This Switches Them Back
            buffer = left_half
            left_half = right_half
            right_half = buffer
        print('Round: ', loop + 1, ' ', bin_to_hex(left_half, 32), ' ',
                bin_to_hex(right_half, 32), ' ', bin_to_hex(key_schedule[loop], 48))
    cyphertext = IP_Inverse(left_half + right_half)
    return cyphertext
```
#### DES Round Function
This is a very important function, it is the core of the Feistel Network that makes up DES. It is called 16 times with different keys from the keyschedule and each time the inputs are mapped to the outputs (after the Feistel Function is called)

```python
def round(left_half, right_half, key):
    new_left_half = xor(left_half, f(right_half, key))
    return (right_half, new_left_half)
```

#### DES Feistel Function

```python
def f(bytes, key):
    expanded = f_expansion(bytes)
    mixed = f_key_mixing(expanded, key)
    substituted = s_boxes(mixed)
    permutation = f_permute(substituted)
    return permutation
```

#### DES Substitution Boxes
These substitution boxes are really the core of the DES algorithm, they split the input up into six bit chunks and map those to 4 output bits this provides the diffusion necessary to make a secure block cipher (not that DES is secure, because it is vulnerable to bruteforce)
```python
def s_boxes(bits):
    # These substitution boxes are really the core of the DES algorithm, they take six input bits and maps it to 4 output bits
    #       this provides the diffusion necessary to make a secure block cipher (not that DES is secure, because it is not)
    sboxes =   [[[14, 4, 13, 1, 2, 15, 11, 8, 3, 10, 6, 12, 5, 9, 0, 7], [0, 15, 7, 4, 14, 2, 13, 1, 10, 6, 12, 11, 9, 5, 3, 8], [4, 1, 14, 8, 13, 6, 2, 11, 15, 12, 9, 7, 3, 10, 5, 0], [15, 12, 8, 2, 4, 9, 1, 7, 5, 11, 3, 14, 10, 0, 6, 13 ]],       
                [[15, 1, 8, 14, 6, 11, 3, 4, 9, 7, 2, 13, 12, 0, 5, 10], [3, 13, 4, 7, 15, 2, 8, 14, 12, 0, 1, 10, 6, 9, 11, 5], [0, 14, 7, 11, 10, 4, 13, 1, 5, 8, 12, 6, 9, 3, 2, 15], [13, 8, 10, 1, 3, 15, 4, 2, 11, 6, 7, 12, 0, 5, 14, 9 ]],
                [[10, 0, 9, 14, 6, 3, 15, 5, 1, 13, 12, 7, 11, 4, 2, 8], [13, 7, 0, 9, 3, 4, 6, 10, 2, 8, 5, 14, 12, 11, 15, 1], [13, 6, 4, 9, 8, 15, 3, 0, 11, 1, 2, 12, 5, 10, 14, 7], [1, 10, 13, 0, 6, 9, 8, 7, 4, 15, 14, 3, 11, 5, 2, 12 ]],
                [[7, 13, 14, 3, 0, 6, 9, 10, 1, 2, 8, 5, 11, 12, 4, 15], [13, 8, 11, 5, 6, 15, 0, 3, 4, 7, 2, 12, 1, 10, 14, 9], [10, 6, 9, 0, 12, 11, 7, 13, 15, 1, 3, 14, 5, 2, 8, 4], [3, 15, 0, 6, 10, 1, 13, 8, 9, 4, 5, 11, 12, 7, 2, 14 ]],
                [[2, 12, 4, 1, 7, 10, 11, 6, 8, 5, 3, 15, 13, 0, 14, 9], [14, 11, 2, 12, 4, 7, 13, 1, 5, 0, 15, 10, 3, 9, 8, 6], [4, 2, 1, 11, 10, 13, 7, 8, 15, 9, 12, 5, 6, 3, 0, 14], [11, 8, 12, 7, 1, 14, 2, 13, 6, 15, 0, 9, 10, 4, 5, 3 ]],
                [[12, 1, 10, 15, 9, 2, 6, 8, 0, 13, 3, 4, 14, 7, 5, 11], [10, 15, 4, 2, 7, 12, 9, 5, 6, 1, 13, 14, 0, 11, 3, 8], [9, 14, 15, 5, 2, 8, 12, 3, 7, 0, 4, 10, 1, 13, 11, 6], [4, 3, 2, 12, 9, 5, 15, 10, 11, 14, 1, 7, 6, 0, 8, 13 ]],
                [[4, 11, 2, 14, 15, 0, 8, 13, 3, 12, 9, 7, 5, 10, 6, 1], [13, 0, 11, 7, 4, 9, 1, 10, 14, 3, 5, 12, 2, 15, 8, 6], [1, 4, 11, 13, 12, 3, 7, 14, 10, 15, 6, 8, 0, 5, 9, 2], [6, 11, 13, 8, 1, 4, 10, 7, 9, 5, 0, 15, 14, 2, 3, 12 ]],
                [[13, 2, 8, 4, 6, 15, 11, 1, 10, 9, 3, 14, 5, 0, 12, 7], [1, 15, 13, 8, 10, 3, 7, 4, 12, 5, 6, 11, 0, 14, 9, 2], [7, 11, 4, 1, 9, 12, 14, 2, 0, 6, 10, 13, 15, 3, 5, 8], [2, 1, 14, 7, 4, 10, 8, 13, 15, 12, 9, 0, 3, 5, 6, 11 ]]]
    # Split the bytes into 8 substrings, split those substrings into the outer and inner bits, 
    #       and use them as indexes for the lookup in the sboxes, convert to binary and pad zeros if necessary
    substituted_string = ''
    for i in range(0,48,6):
        substring = (bits[0:48][i:i+6])
        row = bin_to_int(substring[0] + substring[5])
        column = bin_to_int(substring[1] + substring[2] + substring[3] + substring[4])
        lookup = int_to_bin(sboxes[int(i/6)][row][column])
        if len(lookup) < 4:
            lookup = ((4 - len(lookup)) * '0') + lookup
        substituted_string += lookup
    return substituted_string
```

#### DES Decryption
The decryption of DES is virtually identical to the encryption. The only difference in the code of the two functions is that the key_schedule is read in reverse. This is an inherent property of the Feistel Network. They are invertible. 

Here is the only line that is changed:
```python
left_half, right_half = round(left_half, right_half, key_schedule[15 - loop])
```

#### Key Schedule
Lastly, we have to talk about how DES handles its key generation. 

```python
def generate_keys(intitial_state):

    if len(intitial_state) != 64:
        if len(intitial_state) == 16:
            intitial_state = hex_to_binary(intitial_state, 16)
        else:
            return 'ERROR MESSAGE NOT PROPER SIZE'

    pc_1 = [57, 49, 41, 33, 25, 17, 9, 1, 58, 50, 42, 34, 26, 18, 10, 2, 59, 51, 43, 35, 27, 19, 11, 3, 60, 52, 44, 36, 
            63, 55, 47, 39, 31, 23, 15, 7, 62, 54, 46, 38, 30, 22, 14, 6, 61, 53, 45, 37, 29, 21, 13, 5, 28, 20, 12, 4]
    pc_2 = [14, 17, 11, 24, 1, 5, 3, 28, 15, 6, 21, 10, 23, 19, 12, 4, 26, 8, 16, 7, 27, 20, 13, 2, 41, 
            52, 31, 37, 47, 55, 30, 40, 51, 45, 33, 48, 44, 49, 39, 56, 34, 53, 46, 42, 50, 36, 29, 32 ]
    permuted = permute(intitial_state, pc_1, 56)
    
    key_schedule = []
    left_half, right_half = permuted[0:28], permuted[28:56]
    for loop in range(16):
        if {0,1,8,15}.__contains__(loop):
            left_half, right_half = shifted(left_half, 1), shifted(right_half, 1)
        else:
            left_half, right_half = shifted(left_half, 2), shifted(right_half, 2)
        key_schedule.append(permute((left_half + right_half), pc_2, 48))
    return key_schedule
```

---
### Final Thoughts
DES is conceptually very interesting; it is **the** prototypical block cipher. Implementing it in Python helped to cement the invertiability of the Feistel Structure, and the non-linear transformation that happens when the bits go through the s-boxes. While DES might not be offically secure any longer, it is still incredibly important to the history of cryptography and the basis of the majority of block ciphers. 
