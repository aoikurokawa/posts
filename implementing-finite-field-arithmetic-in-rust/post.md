# Implementing Finite Field Arithmetic in Rust

## Introduction

Finite fields, also known as Galois fields, are algebraic structures that have a finite number of elements and follow specific mathematical rules. They play a fundamental role in various areas of mathematics, computer science, and cryptography.

Significance of Finite Fields in Computer Scienece and Cyrptography:

1. Error Correction and Data Encoding:
- Finite Fields play a crucial role in error-correcting codes, where they are used to encode and decode data to detect and correct during transmission. 

2. Cryptography:
- In modern cryptography, finite fields are used to define elliptic curves, which form the foudation of elliptic curve cryptography (ECC). ECC is widely used in secure communications and digital signatures due to its strong security and efficiency. 
- fintie fields are also used in Bitcoin and other cyrptocurrencies. They use fintie fields as a fundamental building block for several crucial components. 

3. Coding Theory:
- Finite fields are fundamental in coding theory, which deals with error detection and correction codes used in data storage, communication, and compression. 

4. Cryptographic Hash Functions:
- Finite fields are utilized in cryptographic hash functions, which are one-way functions that map data of arbitrary size to fixed-size hash values. These functions are vital for ensuring data integrity and authenticity.

5. Pseudo-random Number Genration:
- Finite fields are used in certain algorithms for generating pseudo-random numbers, which find applications in simulations, games, and cyrptographic protocols.

6. Galois Theory:
- Finite fields are closely connected to Galois theory, a branch of abstract algebra. Galois theory provides insights into the properties of finite and their extensions. 

7. Digital Signal Processing:
- Finite fields are used in certain applications of digital singal processing, such as error correction in audio and image data. 

In this blog, I will try to implement finite fields in Rust. 


## Finte Fields

### Finite Field Definition
Formally, a finite field is defined as a set of elements along with two binary operations, + (addition) and * (multiplication), that satisfy certain properties. The number of elements in a finite field is called its order, denoted by p, where p is a prime number. The finite field is denoted as Fp.

Basic Properties of Finite Fields:

1. Closure: The result of addition and multiplication of any two elements in the finite field also belongs to the finite field. In other words, finite fields are closed under addition and multiplication.

> ex:) If a and b are in the set, a + b and a * b are in the set.

2. Addition: Finite fields form an Abelian group under addition. This means that addition is commutative (a + b = b + a), associative (a + (b + c) = (a + b) + c), and has an identity element (0) such that a + 0 = a for any element a in the field. Additionally, every element has an additive inverse (−a) such that a + (−a) = 0.

3. Multiplication: The non-zero elements of a finite field form an Abelian group under multiplication. This means that multiplication is commutative (a * b = b * a), associative (a * (b * c) = (a * b) * c), and has an identity element (1) such that a * 1 = a for any non-zero element a in the field. Additionally, every non-zero element has a multiplicative inverse (a^−1) such that a * a^−1 = 1.

4. Additive inverse: If a is in the set, -a is in the set, which is defined as the value that makes a + (-a) = 0.

5. Multiplicative inverse: If a is in the set a nd is not 0, a ^ -1 is in the set, which is defined as the value that makes a * a ^ -1 = 1. 


## Finite Field Implementation in Rust

### Construct Field Element
First, we have to create new struct which name is FieldElement.

```rust
#[derive(Debug, Clone)]
pub struct FieldElement {
    pub num: u64,
    pub prime: u64,
}

impl FieldElement {
    pub fn new(num: u64, prime: u64) -> Self {
        
	// check that num is between 0 and prime-1 inclusive. 
	// if not, will be paniced. 
        if num >= prime {
	    panic!("Num {} not in field range 0 to {}", num, prime - 1);
	} 

	Self {
	    num,
	    prime
	}
    }
}

// use PartialEq trait to check whether two fields are equal or not
impl PartialEq for FieldElement {
    fn eq(&self, other: &FieldElement) -> bool {
        self.num == other.num && self.prime == other.prime
    }
}

impl Eq for FieldElement {}


#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_compare_two_field_elements() {
        let a = FieldElement::new(7, 13);
	let b = FieldElement::new(6, 13);

	assert_ne!(a, b);
	assert_eq!(a, a);
    }
}
```

### Mudulo Arithmetic
One of the tools we can use to make a finte field closed under addition, subtraction, multiplication, and division is something called modulo arithmetic. 

> 7 % 3 = 1

I think you have learned in this operation. Whenever the division wasn't even, there was something called the "remainder", which is the amount left over from the actual division.

Let's see some code how to write modulo arithmetic.

In Pyhon:

```python
print(7 % 3)
# 1

print(-27 % 13)
# 12
```

In Rust, we will get another result when calculate with negative number.

```rust
println!("{:?}", (7 % 3));
// 1

println!("{:?}", (-27 % 13));
// -1
```

so we need to use special dependency for taking care of modulo function. 

In this time, I will use [this modulo](https://docs.rs/modulo/latest/modulo/trait.Mod.html) crate.

Cargo.toml
```toml
[dependencies]
modulo = "0.1.2"
```

### Addition and Subtraction
Next, we will add **add** functin and **sub** function.
Rust provides each operation's traits. So we will use for that.

```rust
impl Add for FieldElement {
    type Output = Self;

    fn add(self, rhs: Self) -> Self::Ouput {
        
	// have to ensure that the elements are from the same finite fields
        if self.prime != rhs.prime {
	    panic!("cannot add two numbers in different Fields");
	}
    }

    // addition in a finite field is defined  with the modulo operator
    let num = (self.num + other.num).modulo(self.prime);
    Self {
        num,
	prime: self.prime,
    }
}

```

```rust
impl Sub for FieldElement {
    type Output = Self;

    fn sub(self, rhs: Self) -> Self::Output {
        if self.prime != rhs.prime {
	    panic!("cannot subtract two numbers in different Fields");
	}

	let num = (self.num - other.num).modulo(self.prime);
	Self {
	    num,
	    prime: self.prime,
	}
    }
}


#[cfg(test)]
mod tests {
    use super::*;


...

    #[test]
    fn test_add() {
        let a = FieldElement::new(7, 13);
	let b = FieldElement::new(12, 13);
	let c = FieldElement::new(6, 13);

	assert_eq!(a + b, c);
    }
}
```

### Multiplication and Exponension

```rust
impl Mul for FieldElement {
    type Output = Self;

    fn mul(self, rhs: Self) -> Self::Output {
        if self.prime != rhs.prime {
	    panic!("cannot subtract two numbers in different Fields");
	}

	let num = (self.num * other.num).modulo(self.prime);
	Self {
	    num,
	    prime: self.prime,
	}
    }
}

#[cfg(test)]
mod tests {
    use super::*;


...

    #[test]
    fn test_mul() {
        let a = FieldElement::new(3, 13);
	let b = FieldElement::new(12, 13);
	let c = FieldElement::new(10, 13);

	assert_eq!(a * b, c);
    }
}
```

Since I guess Rust does not have exponension trait, so we will define **pow** function inside impl FiniteField {}.

```rust
impl FieldElement {

...

    pub fn pow(&self, exponent: u32) -> Self {
        let num = self.num.pow(exponent).modulo(self.prime);
	Self {
	    num,
	    prime: self.prime,
	}
    }

}

#[cfg(test)]
mod tests {
    use super::*;


...

    #[test]
    fn test_pow() {
        let a = FieldElement::new(3, 13);
	let c = FieldElement::new(1, 13);

	assert_eq!(a.pow(3), c);
    }
}
```

### Division
When it comes to division of finite field, it is a bit difficult to understand for the first time. 
We can implement like this.

```rust
impl Div for FieldElement {
    type Output = Self;

    fn div(self, rhs: Self) -> Self::Output {
        if self.prime != rhs.prime {
            panic!("cannot divide two numbers in different Fields");
        }

        // use Fermat's little theorem
        // self.num.pow(p-1) % p == 1
        // this means:
        // 1/n == pow(n, p-2, p) in Python
        let exp = rhs.prime - 2;
        let num_pow = rhs.pow(exp);
        let result = self.num * num_pow.num;
        Self {
            num: result % self.prime,
            prime: self.prime,
        }
    }
}


#[cfg(test)]
mod tests {
    use super::*;


...

    #[test]
    fn test_div() {
        let prime = 19;
        let mut a = FieldElement::new(2, prime);
        let mut b = FieldElement::new(7, prime);
        let mut c = FieldElement::new(3, prime);

        assert_eq!(a / b, c);

        a = FieldElement::new(7, prime);
        b = FieldElement::new(5, prime);
        c = FieldElement::new(9, prime);

        assert_eq!(a / b, c);
    }
}
```

That's all for all operations. If you get lost, you can see the whole code [here](https://github.com/aoikurokawa/rueth/blob/master/finite-fields/src/lib.rs).


## Conclusion
In this blog, we learned about finte fields and how to implement them in Rust. I will use FieldElement struct in next blog. 
See you in next blog.
