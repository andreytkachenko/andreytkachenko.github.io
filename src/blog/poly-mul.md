# Point-value / Coefficient forms

Here is a showcase of multiplying two polynomials in point-value form:

 * 1st polynomial: 1x^2 + 1
 * 2nd polynomial: 2x^2 + 2
 * 3rd polynomial (what should we get by multiplying the above two): 2x^4 + 4x^2 + 2

The resulting polynomial is of degree 4. So, we will need at least 5 points to be able to represent that polynomial in point-value form. 

Let’s pick 5 points each from our 1st and 2nd polynomial.

  *  5 points for polynomial 1x^2+1:
      -  (0, 1), (1, 2), (2, 5), (3, 10), (4, 17)
  *  5 points for polynomial 2x^2+2:
      -  (0, 2), (1, 4), (2, 10), (3, 20), (4, 34)

Let’s multiply the y values of the points that share the same x value.
```
   (0,1) . (0,2)  = (0,2)
   (0,1) . (0,2)  = (0,2)
   (1,2) . (1,4)  = (1,8)
   (1,2) . (1,4)  = (1,8)
   (2,5) . (2,10) = (2,50)
   (2,5) . (2,10) = (2,50)
  (3,10) . (3,20) = (3,200)
  (3,10) . (3,20) = (3,200)
  (4,17) . (4,34) = (4,578)
  (4,17) . (4,34) = (4,578)
```
If you plug these points we just calculated, into the 3rd polynomial, you’ll see they will satisfy the equation.

## How will this be useful for the regular multiplication?

### Here is the recipe:

  1. Have 2 numbers to multiply
  2. Treat these numbers as the coefficient representation of a polynomial
  3. Compute the point-value form of these 2 polynomials
  4. Multiply these 2 polynomials in point-value form
  5. Convert the resulting polynomial back into coefficient form
  6. Here is the result of your multiplication!
  
### A demo for the above recipe:

I will use the same example from our multiplication in point-value form, so that it would be easier to follow:

  1. 11 and 22
  2. 1x^2 + 1 for 11
  3. 2x^2 + 2 for 22
  4. (0, 1), (1, 2), (2, 5), (3, 10), (4, 17) for 11
     (0, 2), (1, 4), (2, 10), (3, 20), (4, 34) for 22
  5. Point-value form: (0, 2), (1, 8), (2, 50), (3, 200), (4, 578)
  6. Coefficient form: 2x^4 + 4x^2 + 2
  7. 242 = 11 x 22 indeed!

Our astute readers may realize that we may have reduced the complexity of multiplication to N from N^2, but now we have another problem. The complexity of computing the point-value form from the coefficient form is N^2. 

## Why decomposing a wave and transitioning between coefficient and point-value forms are similar?

At first glance, decomposing a wave and transitioning between coefficient to point-value form, seem to be very unrelated. But if they share the exact same formula, they should be sharing a very common intuition behind all these.

I have a great answer on why `decomposing a wave` and `transitioning between coefficient to point-value form` are nearly identical.

Recall these transformations from `point-value form` to `coefficient form` from above:

  * Point-value: (0, 2), (1, 8), (2, 50), (3, 200), (4, 578)
  * Coefficient: 2x^4 + 4x^2 + 2

What we are doing is, inputting some points from the polynomial, and outputting the formula of the polynomial.

We can say the same for decomposing waves. We are inputting some data points from the wave, and getting back the formula of the wave (amplitudes and phases of the sine waves in it).

Of course, the above explanation is an oversimplification that misses some details. But it really is the intuition behind all these:

```python
import numpy
from numpy.fft import fft, ifft


def poly_mul(p1, p2):
    """Multiply two polynomials.

    p1 and p2 are arrays of coefficients in degree-increasing order.
    """
    deg1 = p1.shape[0] - 1
    deg2 = p1.shape[0] - 1
    # Would be 2*(deg1 + deg2) + 1, but the next-power-of-2 handles the +1
    total_num_pts = 2 * (deg1 + deg2)
    next_power_of_2 = 1 << (total_num_pts - 1).bit_length()

    ff_p1 = fft(numpy.pad(p1, (0, next_power_of_2 - p1.shape[0])))
    ff_p2 = fft(numpy.pad(p2, (0, next_power_of_2 - p2.shape[0])))
    product = ff_p1 * ff_p2
    inverted = ifft(product)
    rounded = numpy.round(numpy.real(inverted)).astype(numpy.int32)
    return numpy.trim_zeros(rounded, trim='b')


p1 = numpy.array([1, 0, 1])
p2 = numpy.array([2, 0, 2])
print(poly_mul(p1, p2))
```

