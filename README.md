# RealFFT: Real-to-complex FFT and complex-to-real iFFT based on RustFFT

This library is a wrapper for RustFFT that enables faster computations when the input data is real.
It packs a 2N long real vector into an N long complex vector, which is transformed using a standard FFT.
It then post-processes the result to give only the first half of the complex spectrum, as an N+1 long complex vector.

The iFFT goes through the same steps backwards, to transform an N+1 long complex spectrum to a 2N long real result.

The speed increase compared to just converting the input to a 2N long complex vector
and using a 2N long FFT depends on the length f the input data.
The largest improvements are for long FFTs and for lengths over around 1000 elements there is an improvement of about a factor 2.
The difference shrinks for shorter lengths, and around 100 elements there is no longer any difference.  

## Why use real-to-complex fft?
### Using a complex-to-complex fft
A simple way to get the fft of a rea values vector is to convert it to complex, and using a complex-to-complex fft.

Let's assume `x` is a 6 element long real vector: 
```text
x = [x0r, x1r, x2r, x3r, x4r, x5r]
```

Converted to complex, using the notation `(xNr, xNi)` for the complex value `xN`, this becomes: 
```text
x_c = [(x0r, 0), (x1r, 0), (x2r, 0), (x3r, 0), (x4r, 0, (x5r, 0)]
```


The general result of `X = FFT(x)` is:
```text
X = [(X0r, X0i), (X1r, X1i), (X2r, X2i), (X3r, X3i), (X4r, X4i), (X5r, X5i)]
```

However, because our `x` was real-valued, some of this is redundant:
```text
FFT(x_c) = [(X0r, 0), (X1r, X1i), (X2r, X2i), (X3r, 0), (X2r, -X2i), (X1r, -X1i)]
```

As we can see, the output contains a fair bit of redundant data. But it still takes time for the FFT to calculate these values. Converting the input data to complex also takes a little bit of time.

### real-to-complex
Using a real-to-complex fft removes the need for converting the input data to complex.
It also avoids caclulating the redundant output values.

The result is: 
```text
RealFFT(x) = [(X0r, 0), (X1r, X1i), (X2r, X2i), (X3r, 0)]
```

This is the data layout output by the real-to-complex fft, and the one expected as input to the complex-to-real ifft.

## Scaling
RealFFT matches the behaviour of RustFFT and does not normalize the output of either FFT of iFFT. To get normalized results, each element must be scaled by `1/sqrt(length)`. If the processing involves both an FFT and an iFFT step, it is advisable to merge the two normalization steps to a single, by scaling by `1/length`.

## Documentation

The full documentation can be generated by rustdoc. To generate and view it run:
```text
cargo doc --open
```

## Benchmarks

To run a set of benchmarks comparing real-to-complex FFT with standard complex-to-complex, type:
```text
cargo bench
```
The results are printed while running, and are compiled into an html report containing much more details.
To view, open `target/criterion/report/index.html` in a browser.

## Example
Transform a vector, and then inverse transform the result.
```rust
use realfft::{ComplexToReal, RealToComplex};
use rustfft::num_complex::Complex;
use rustfft::num_traits::Zero;

// make dummy input vector, spectrum and output vectors
let mut indata = vec![0.0f64; 256];
let mut spectrum: Vec<Complex<f64>> = vec![Complex::zero(); 129];
let mut outdata: Vec<f64> = vec![0.0; 256];

//create an FFT and forward transform the input data
let mut r2c = RealToComplex::<f64>::new(256).unwrap();
r2c.process(&mut indata, &mut spectrum).unwrap();

// create an iFFT and inverse transform the spectum
let mut c2r = ComplexToReal::<f64>::new(256).unwrap();
c2r.process(&spectrum, &mut outdata).unwrap();
```

## Compatibility

The `realfft` crate requires rustc version 1.37 or newer.