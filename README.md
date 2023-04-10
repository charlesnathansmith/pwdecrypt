# pwcrack
This program demonstrates a flaw in a password encryption method used by a commercial product

pwencrypt.h contains the original routine as reverse engineered from the product binary

pwcrack.cpp demonstrates a known plaintext attack that can completely circumvent the offered protection

The original encryption method hashes the provided password in order to generate two shift registers that are then used to actually encrypt the data.

While it would be difficult to recover the orignal password, determining the generated shift registers is enough to fully encrypt or decrypt any data, and these are relatively trivial to recover from a small sample of corresponding plaintext and encrypted values

The main encryption loop we are interested in looks like this:

```
for (size_t i = 0; i < len; i++)
{
    crypt_out[i] = plain_in[i] ^ shift_a;

    shift_b = rol32(shift_b, shift_a & 0xff);
    shift_a ^= shift_b;

    shift_a = ror32(shift_a, (shift_b >> 8) & 0xff);
    shift_b += shift_a;
}
```

Where shift_a and shift_b are unknown values generated by hashing the password

We can rename the intermediates to clarify which calculated values we are discussing:

```
for (size_t i = 0; i < len; i++)
{
    crypt_out[i] = plain_in[i] ^ shift_a1;

    shift_b2 = rol32(shift_b1, shift_a1 & 0xff);
    
    shift_a2 = shift_a1 ^ shift_b2;
    shift_a3 = ror32(shift_a2, (shift_b2 >> 8) & 0xff);
    
    shift_b3 += shift_a3;
}
```

Since we have pairs of plaintext and encrypted data, we can calculate the following outright

```
shift_a1 = crypt_out[0] ^ plain_in[0];
shift_a3 = crypt_out[1] ^ plain_in[1];
```

Examining the encryption line by line:

```
shift_b2 = rol32(shift_b1, shift_a1 & 0xff);
```

We don't care about this at all for now.  shift_b1 was unknown, so rotating it just gives shift_b2 as another unknown

```
shift_a2 = shift_a1 ^ shift_b2;
shift_a3 = ror32(shift_a2, (shift_b2 >> 8) & 0xff);
```

Since we know shift_a1 and shift_a2, and only one byte is being used to determine the rotation, it is trivial to exhaustively search for values of shift_b2 that will convert shift_a1 into shift_a3

We just undo the rotation by each possible value and see if the rotation amount matches (shift_b2 >> 8) & 0xff

The number of shift_b2 values that satisfy this requirement will vary, but there were only 4 found in the provided example, and the max should probably be something like 8 or 32 due to rotations past 32 bits repeating.  I haven't worked out the exact number, but the hard limit is 256 which could still be quickly evaluated

Once we have a candidate shift_b2 value, then we can worry about undoing the rotation that generated it earlier to get back a candidate shift_b1 value

```
std::set<uint32_t> b_test;

for (size_t i = 0; i <= 0xff; i++)
    {
        uint32_t shift_a2 = rol32(shift_a3, i);
        uint32_t shift_b2 = shift_a2 ^ shift_a1;

        if (((shift_b2 >> 8) & 0xff) == i)
        {
            uint32_t shift_b1 = ror32(shift_b2, shift_a1 & 0xff);
            b_test.insert(shift_b1);
        }
    }
```

Now all that's left to do is to try to encrypt the plaintext with each candidate value until we find one that succeeds

```
for (uint32_t _b : b_test)
{
    uint32_t a = plain[0] ^ crypt[0];
    uint32_t b = _b;
    bool found = true;

    for (size_t i = 0; i < len; i++)
    {
        if ((plain[i] ^ a) != crypt[i])
        {
            found = false;
            break;
        }

        b = rol32(b, a & 0xff);
        a ^= b;
        a = ror32(a, (b >> 8) & 0xff);
        b += a;
    }

    if (found)
    {
        uint32_t shift_a = plain[0] ^ crypt[0];
        uint32_t shift_b = _b;
 
        printf("shift_a: %.8x\n", *shift_a);
        printf("shift_b: %.8x\n\n", *shift_b);
        return true;
    }
}
```

A routine that can encrypt and decrypt data using the recovered shift values without the password is also included in pwcrack.cpp

Just to emphasize, this wasn't a toy example, but a real encryption routine being used in a commercial product.  A lot of effort was put into making the password unrecoverable, but you end up not needing it to recover the values that actually matter.

We've been assuming here that we have the first few pairs of plaintext and encrypted values generated by this routine, but even if we only recovered some of them mid-stream, then as long as we know what position in the stream they come from, it is possible to use the same method to solve the shift registers used to encode the values at that position, then deterministically unroll the operations performed on them at each position until we recover that starting shift register values, though that it not demonstrated in the example code
