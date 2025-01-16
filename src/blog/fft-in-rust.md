```rust
use std::f64::consts::TAU;

// simple O(n^2) variant
fn main1() {
    let data = vec![0.0, 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0];
    let n = data.len();
    
    let mut result = vec![(0.0, 0.0); n]; // Real and imaginary parts
    
    for k in 0..n {
        let mut real_sum = 0.0;
        let mut imag_sum = 0.0;
        
        for t in 0..n {
            let angle = std::f64::consts::TAU * (t as f64) * (k as f64) / (n as f64);

            real_sum += data[t] * angle.cos();
            imag_sum -= data[t] * angle.sin();
        }
        
        result[k] = (real_sum, imag_sum);
    }
    
    println!("DFT Result: {:?}", result);
}

/// Cooley-Tukey Variant
///   input.len() must be the power of two
fn fft(input: &mut [Complex], inverse: bool) {
    let n = input.len();
    let angle = if inverse { TAU } else {-TAU};

    // Bit-reversal permutation
    let mut j = 0;
    for i in 1..n - 1 {
        let mut m = n >> 1;

        while j >= m {
            j -= m;
            m >>= 1;
        }

        j += m;
        if j > i {
            input.swap(i, j);
        }
    }

    // Iteratively compute the FFT
    let mut step = 2;
    while step <= n {
        for k in (0..n).step_by(step) {
            let half_step = step / 2;

            let w_theta = angle / step as f64;
            let mut w = Complex::new(1.0, 0.0);
            let (sin, cos) = w_theta.sin_cos();
            let w_step = Complex::new(cos, sin);

            for m in 0..half_step {
                let t = input[k + m + half_step] * w;
                let u = input[k + m];

                input[k + m] = u + t;
                input[k + m + half_step] = u - t;
                
                w *= w_step;
            }
        }
        step *= 2;
    }

    if inverse {
        for i in input {
            i.real /= n as f64;
            i.imag /= n as f64;
        }
    }
}

#[derive(Debug, Clone, Copy)]
struct Complex {
    real: f64,
    imag: f64,
}

impl std::fmt::Display for Complex {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{} {}j", self.real, self.imag)
    }
}

impl Complex {
    fn new(real: f64, imag: f64) -> Self {
        Complex { real, imag }
    }
}

impl std::ops::Add for Complex {
    type Output = Complex;

    fn add(self, other: Complex) -> Complex {
        Complex {
            real: self.real + other.real,
            imag: self.imag + other.imag,
        }
    }
}

impl std::ops::Sub for Complex {
    type Output = Complex;

    fn sub(self, other: Complex) -> Complex {
        Complex {
            real: self.real - other.real,
            imag: self.imag - other.imag,
        }
    }
}

impl std::ops::Mul for Complex {
    type Output = Complex;

    fn mul(self, other: Complex) -> Complex {
        Complex {
            real: self.real * other.real - self.imag * other.imag,
            imag: self.real * other.imag + self.imag * other.real,
        }
    }
}

impl std::ops::MulAssign for Complex {
    fn mul_assign(&mut self, other: Complex) {
        let real = self.real * other.real - self.imag * other.imag;
        let imag = self.real * other.imag + self.imag * other.real;

        self.real = real;
        self.imag = imag;
    }
}

fn main() {
    // Example usage
    let mut input = [
        Complex::new(0.0, 0.0),
        Complex::new(1.0, 0.0),
        Complex::new(2.0, 0.0),
        Complex::new(3.0, 0.0),
        Complex::new(4.0, 0.0),
        Complex::new(5.0, 0.0),
        Complex::new(6.0, 0.0),
        Complex::new(7.0, 0.0),
    ];

    fft(&mut input, false);
    fft(&mut input, true);

    for c in &input {
        println!("{c}");
    }
}
```
