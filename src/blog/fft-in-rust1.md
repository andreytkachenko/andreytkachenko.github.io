## Простой агоритм FFT на Rust

```rust
use num_complex::Complex;
use num_traits::Float;
use std::f64::consts::TAU;

fn sort_values<T: Copy>(dst: &mut [Complex<T>], src: &[Complex<T>]) {
    let bit_size = usize::BITS - src.len().trailing_zeros();

    for i in 0..src.len() {
        dst[i.reverse_bits() >> bit_size] = src[i];
    }
}

fn fft<T: Copy + Clone + Float>(input: &[Complex<T>], output: &mut [Complex<T>], inverse: bool) {
    let n = input.len();
    assert!(n.is_power_of_two());

    let n_bf = n / 2;
    let n_stages = n.trailing_zeros();

    let coeff = (0..n).map(|x| x * n_bf).collect::<Vec<_>>();
    let angle = T::from(if inverse { TAU } else { -TAU }).unwrap();

    let theta = angle / T::from(n).unwrap();
    let w_n = Complex::new(theta.cos(), theta.sin());

    sort_values(output, input);

    for i in 0..n_stages {
        let h = 1 << i;
        let stage_coeff = coeff.iter().copied().map(|c| c / h).collect::<Vec<_>>();

        for j in 0..n_bf {
            let k = (h * (j / h)) * 2 + (j % h);

            let w_coeff_a = stage_coeff[k] % n;
            let w_coeff_b = stage_coeff[k + h] % n;

            let w_a = w_n.powi(w_coeff_a as _);
            let w_b = w_n.powi(w_coeff_b as _);

            // butterfly
            let out_k = output[k];
            let out_kh = output[k + h];

            output[k] = out_k + w_a * out_kh;
            output[k + h] = w_b * out_kh + out_k;
        }
    }

    if inverse {
        for x in output.iter_mut() {
            *x = x.unscale(T::from(n).unwrap());
        }
    }
}

fn main() {
    // Создание примера сигнала
    let size = 8;
    let mut signal: Vec<Complex<f64>> = (0..size).map(|x| Complex::new(x as _, 0.0)).collect();
    let mut signal2 = signal.clone();

    println!("Исходный сигнал:");
    for val in &signal {
        print!("{:?} ", val);
    }
    println!("\n");

    fft(&signal, &mut signal2, false);

    println!("Сигнал после БПФ:");
    for val in &signal2 {
        print!("{:?} ", val);
    }
    println!();

    fft(&signal2, &mut signal, true);

    println!("Сигнал после iБПФ:");
    for val in &signal {
        print!("{:?} ", val);
    }
    println!();
}
```
