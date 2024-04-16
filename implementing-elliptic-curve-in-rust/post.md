# Implementing Elliptic Curves in Rust: A Comprehensive Guide


![Screenshot from 2023-08-27 11-05-14](https://github.com/aoikurokawa/blog/assets/62386689/50a59867-69f3-4104-ba93-fb51ad1caff7)


## Introduction

In the [last blog](https://www.nxted.co.jp/blog/blog_detail?id=21), we learned about finite field and implemented it in Rust programming language.   
In this blog post, we are going to learn Elliptic Curves. Like last blog, I will explain the defintion of elliptic curve and will implement in Rust programming language. 
Let's get started!


## Understanding Elliptic Curve

An elliptic curve is a mathematical concept that plays a crucial role in various fields, including cryptography and number theory. It is a specific type of curve defined by an equation of the form:
y² = x³ + ax + b

Here, "a" and "b" are constants that define the characteristics of the curve. The curve is plotted on a two-dimensional plane with an x-axis and a y-axis. The interesting and useful property of elliptic curves is their ability to form a group structure with a particular point called the "point at infinity" acting as the identity element.

Elliptic curves have several intriguing properties:

- Symmetry: An elliptic curve is symmetric with respect to the x-axis, meaning that if a point (x, y) lies on the curve, the point (x, -y) also lies on the curve.

- Cubic Nature: The equation of an elliptic curve involves cubic terms, which give the curve its distinct shape.

- Multiple Intersections: An elliptic curve can intersect the x-axis at one, two, or three points, including the point at infinity. This property is used in cryptographic applications.

- Group Structure: The set of points on an elliptic curve, along with an operation called "point addition," forms an additive abelian group. This property is fundamental in cryptographic applications, particularly in elliptic curve cryptography (ECC).

Elliptic curve cryptography (ECC) leverages the difficulty of solving certain mathematical problems involving elliptic curves for its security. One of the most well-known uses of ECC is in public key cryptography, where the security of communication relies on the difficulty of solving the elliptic curve discrete logarithm problem. For more detail, I will try to explain in the next blog. 


## Coding Elliptic Curves in Rust

### Define a Point struct

```rust
#[derive(Debug, Clone)]
pub struct Point {
    pub x: i32,
    pub y: i32,
    pub a: i32,
    pub b: i32,
}
```

### Implement two methods: `new` and `equal`

```rust
impl Point {
    pub fn new(x: i32, y: i32, a: i32, b: i32) -> Self {
        if y.pow(2) != x.pow(3) + (a * x) + b {
            panic!("({:?}, {:?}) is not on the curve", x, y);
        }

        Self { a, b, x, y }
    }

    pub fn equal(&self, rhs: Option<Point>) -> bool {
        *self == rhs.unwrap()
    }
}
```

Let's check equal method is working or not. 

```rust
#[cfg(test)]
mod tests {
  use super::*;

  #[test]
  fn test_equal() {
        let point1 = Point::new(-1, -1, 5, 7);
        let point2 = Point::new(-1, -1, 5, 7);

        assert!(point1.equal(Some(point2)));
  }
}
```


## Point Addition
Point addition is a fundamental operation in elliptic curve mathematics, and it forms the basis for various cryptographic algorithms. It allows you to combine two points on an elliptic curve to produce a third point that also lies on the curve. This operation is defined in a way that respects the group structure of the elliptic curve.
Point addition can be conceptualized based on the observation that lines intersect the elliptic curve either once or three times. When two points are used to define a line, this line intersects the curve again, resulting in a third point. This third point is derived from the reflection of the intersection point over the x-axis, illustrating the outcome of the point addition process.

> P1 = (x1, y1) and p2 = (x2, y2), we get P1 + P2 as follows:
> - Find the point intersecting the elliptic curve a third time by drawing a line through P1 and P2.
> - Reflect the resulting point over the x-axis

it looks like this
![Screenshot from 2023-08-27 11-13-34](https://github.com/aoikurokawa/blog/assets/62386689/a1fd5a2f-0719-42d2-8a90-7cb78b56cca4)

Initially, we draw a line connecting the two points being added, namely A and B. The point where this line intersects the curve for the third time is designated as C. Following this, we mirror this intersection point over the x-axis, effectively arriving at the result of A + B in the above image.

I recommend watching some videos explaing about point addition.

[![point addition](https://markdown-videos-api.jorgenkh.no/url?url=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3DdCvB-mhkT0w)](https://www.youtube.com/watch?v=dCvB-mhkT0w)

## Math of Point Addition
Point addition possesses specific properties that align with our understanding of addition, including:

- Identity
- Commutativity
- Associativity
- Invertiblity

`Identity` means that there's a zero. That is, there exists some point I that, when added to a point A, results in A:

> I + A = A

We will call this point the **point at infinity**

This is related to invertibility. For some point A, there's some other point -A that results i the identity point. That is:

> A + (-A) = I

These points are opposite one another over the x-axis on the curve. 

![Screenshot from 2023-08-27 11-31-12](https://github.com/aoikurokawa/blog/assets/62386689/88f1e2d8-13ef-4ec5-ba96-e4b4f43f76d2)

Hence, we refer to this point as the "point at infinity." The elliptic curve encompasses an additional point, leading to the vertical line intersecting the curve for a third instance.

## Coding Point Addition 
In order to code point addition, we need to change a bit structure of Point. 

```rust
#[derive(Debug, PartialEq, Copy, Clone)]
pub struct Point {
    pub x: Option<i32>,
    pub y: Option<i32>,
    pub a: i32,
    pub b: i32,
}
```

also new method...

```rust
impl Point {
    pub fn new(x: Option<i32>, y: Option<i32>, a: i32, b: i32) -> Self {
            // The x coordinate and y coordinate being None is how we signify the point at infinity.
            if y.unwrap().pow(2) != x.unwrap().pow(3) + (a * x.unwrap()) + b {
                panic!("({}, {}) is not on the curve", x.unwrap(), y.unwrap());
            }

        Self { a, b, x, y }
    }

...
}

```

Implements Add trait for Point
```rust
impl Add for Point {
    type Output = Self;

    fn add(self, rhs: Self) -> Self::Output {
        if self.a != rhs.a || self.b != rhs.b {
            panic!("Points {:?}, {:?} are not on the same curve", self, rhs);
        }

        match (self.x, rhs.x) {
            // Handle the case where the two points are additive inverses (that is, they have the same x but a different y, causing a vertical line)
            // This should return the point at infinity
            (Some(self_x), Some(rhs_x)) if self_x == rhs_x && self.y != rhs.y => {
                return Self {
                    x: None,
                    y: None,
                    a: self.a,
                    b: self.b,
                }
            }
            // self.x being None means that self is the point is the point at infinity, or the additive identity. Thus, we return rhs. 
            (None, Some(_)) => return rhs,
            // rhs.x being None means that rhs is the point at infinity, or the additive identity. Thus, we return self.
            (Some(_), None) => return *self,
            _ => Self {
                x: self.x,
                y: self.y,
                a: self.a,
                b: self.b,
            }
        }
    }
}
```

## Point addition for when x1 != x2
Having discussed the concept of the vertical line, let's now delve into scenarios where the points are distinct. In situations where the x-coordinates of the points differ, we can perform addition using a relatively straightforward equation. To enhance our understanding, let's begin by determining the slope formed by these two points. We can compute this slope using a formula commonly found in pre-algebra:


> P1 = (x1,y1), P2 = (x2,y2), P3 = (x3,y3)
> P1 + P2 = P3
> s = (y2 – y1)/(x2 – x1)

The calculated slope serves as a crucial factor in determining x3. Once we have determined the value of x3, the corresponding y3 can then be computed. The formula to derive P3 is as follows:

> x3 = s2 – x1 – x2
> y3 = s(x1 – x3) – y1

```rust
        match (self.x, rhs.x) {
            // s = (y2 – y1)/(x2 – x1)
            // x3 = s2 – x1 – x2
            // y3 = s(x1 – x3) – y1
            (Some(self_x), Some(rhs_x)) if self_x != rhs_x => {
                let s = (rhs.y.unwrap() - self.y.unwrap()) / (rhs.x.unwrap() - self.x.unwrap());
                let x3 = s.pow(2) - self.x.unwrap() - rhs.x.unwrap();
                let y3 = s * (self.x.unwrap() - x3) - self.y.unwrap();
                return Self {
                    x: Some(x3),
                    y: Some(y3),
                    a: self.a,
                    b: self.b,
                };
            }

            ...
        }
```

## Point Addition for when P1 = P2
In scenarios where the x-coordinates are identical but the y-coordinate differs, a situation arises where the points are positioned symmetrically across the x-axis from each other. In this case, it can be inferred that:

> P1 = –P2 or P1 + P2 = I

We have already handled this in Exercise 3. 

When P1 is equal to P2, a visual representation indicates the need to calculate the tangent line to the curve at point P1. Subsequently, the point of intersection between this tangent line and the curve must be determined.

Looks like this...

![Screenshot from 2023-08-27 12-04-30](https://github.com/aoikurokawa/blog/assets/62386689/319a1cc9-15c8-4db0-b327-fb6fa1e53eef)

Once again, we'll find the slope of the tangent point:

> P1 = (x1,y1), P3 = (x3,y3)
> P1 + P1 = P3
> s = (3x12 + a)/(2y1)

> x3 = s2 – 2x1
> y3 = s(x1 – x3) – y1

```rust
        match (self.x, rhs.x) {
            // s = (3x12 + a)/(2y1)
            // x3 = s2 – 2x1
            // y3 = s(x1 – x3) – y1
            (Some(self_x), Some(rhs_x)) if self_x == rhs_x && self.y == rhs.y => {
                let s = (3 * self_x.pow(2) + self.a) / (2 * self.y.unwrap());
                let x3 = s.pow(2) - 2 * self_x;
                let y3 = s * (self_x - x3) - self.y.unwrap();
                Self {
                    x: Some(x3),
                    y: Some(y3),
                    a: self.a,
                    b: self.b,
                }
            }

            ...
        }
```

the whole code inside fn add() is [here](https://github.com/aoikurokawa/rueth/commit/24274af760620590cc132e64a8ff86028bc7d65b?diff=split#diff-92dac1365e4e6e7496b244cf8fb237375766974272897d0096991c55002157a9R29).

## Conclusion

I have covered a lot of things about elliptic curves. But it would be really important concept for next blog. In the next blog, I will combine two blogs finte fields and elliptic curves. 
Anyway than you for reading. See you in next blog. 

![rustcean](http://i6.glitter-graphics.org/pub/1201/1201566f58nntlu0y.gif)
