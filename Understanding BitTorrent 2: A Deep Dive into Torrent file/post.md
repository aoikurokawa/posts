## Introduction 
BitTorrent is one of the pioneering and most popular peer-to-peer (P2P) file sharing protocols. In the [last blog](https://www.nxted.co.jp/blog/blog_detail?id=40), I wrote about [Bencode](https://en.wikipedia.org/wiki/Bencode). Bencode is the encoding method used by BitTorrent for storing and transmitting loosely structured data. It's a binary format that serializes data types like integers, strings, lists, and dictionaries (key-value pairs). In this blog, we will try to parse a .torrent file.


## Bittorrent file

A .torrent file describes the contents of a torrentable file and information for connecting to a tracker. For example, Debian’s .torrent file looks like this:

```
d
  8:announce
    41:http://bttracker.debian.org:6969/announceDebian’s .torrent file looks like this:
  7:comment
    35:"Debian CD from cdimage.debian.org"
  13:creation date
    i1573903810e
  4:info
    d
      6:length
        i351272960e
      4:name
        31:debian-10.2.0-amd64-netinst.iso
      12:piece length
        i262144e
      6:pieces
        26800:�����PS�^�� (binary blob of the hashes of each piece)
    e
e
```

## Define a `Torrent` struct

First of all, we need to define the struct of Torrent. According to the [spec](https://www.bittorrent.org/beps/bep_0003.html), there are `announce` attribute and some information in `info` attribute. 

> metainfo files
>
> Metainfo files (also known as .torrent files) are bencoded dictionaries with the following keys:
>
> announce
>    The URL of the tracker.
> info
>    This maps to a dictionary, with keys described below.
>
> All strings in a .torrent file that contains text must be UTF-8 encoded.

```rs
// torrent.rs

use serde::{Deserialize, Deserializer, Serialize};

/// A Metainfo files (also known as .torrent files).
#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct Torrent {
    /// The URL of the tracker
    pub announce: String,

    pub info: Info,
}
```

## Define a Info struct

Inside `info` attribute in Torrent struct, there are some attributes more like below explanation. So we need to define struct of `Info`.

> info dictionary
>
> The name key maps to a UTF-8 encoded string which is the suggested name to save the file (or directory) as. It is purely advisory.
> 
> piece length maps to the number of bytes in each piece the file is split into. For the purposes of transfer, files are split into fixed-size pieces which are all the same length except for possibly the last one which may be truncated. piece length is almost always a power of two, most commonly 2 18 = 256 K (BitTorrent prior to version 3.2 uses 2 20 = 1 M as default).
> 
> pieces maps to a string whose length is a multiple of 20. It is to be subdivided into strings of length 20, each of which is the SHA1 hash of the piece at the corresponding index.
> 
> There is also a key length or a key files, but not both or neither. If length is present then the download represents a single file, otherwise it represents a set of files which go in a directory structure.
> 
> In the single file case, length maps to the length of the file in bytes.
> 
> For the purposes of the other keys, the multi-file case is treated as only having a single file by concatenating the files in the order they appear in the files list. The files list is the value files maps to, and is a list of dictionaries containing the following keys:
> 
> length - The length of the file, in bytes.
> 
> path - A list of UTF-8 encoded strings corresponding to subdirectory names, the last of which is the actual file name (a zero length list is an error case).
> 
> In the single file case, the name key is the name of a file, in the muliple file case, it's the name of a directory.


```rs
// torrent.rs

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct Info {
    /// The suggested name to save the file (or directory) as. It is purely advisory.
    ///
    /// In the single file case, the name key is the name of a file, in the multiple file case,
    /// it's the nmae of a directory.
    pub name: String,

    /// The number of bytes in each piece the file is split into.
    ///
    /// For the purposes of transfer, files are split into fixed-size pieces which are all the same
    /// length except for possibly the last one which may be truncated. piece length is almost
    /// always a power of two, most commonly 2^18 = 256K
    /// (BitTorrent prior to version 3.2 uses 2^20 = 1 M as default).
    #[serde(rename = "piece length")]
    pub plength: usize,

    /// Each of which is the SHA1 hash of the piece at the corresponding index.
    pub pieces: Hashes,

    #[serde(flatten)]
    pub keys: Keys,
}
```

## Understanding pieces in a Torrent File

`name` and `piece length` is relatively easy to define a type. However, `pieces` and `keys` are a bit tricky.
In a BitTorrent file, pieces play a crucial role. Think of the entire file you're downloading via BitTorrent as a puzzle. Each piece of this puzzle is a chunk of the file, uniquely identified by a SHA1 hash. The pieces field in the torrent file is a long string containing all these hashes, one after another. Each hash is exactly 20 bytes long. Our task is to correctly serialize (convert to a suitable format for storage or transfer) and deserialize (convert back to the original format) these hashes.


### 1. The Hashes Struct

```rs
#[derive(Debug, Clone)]
pub struct Hashes(pub Vec<[u8; 20]>);
```

This struct represents the `pieces` in a structured format. It's a vector where each element is a 20-byte array, corresponding to one SHA1 hash.

### 2. The Visitor Pattern

In Rust, the Visitor pattern is a way to abstract the logic needed for deserialization. Here are some expample from [doc](https://serde.rs/custom-serialization.html). Our `HashesVisitor` will guide how to turn the raw bytes into our Hashes struct.

```rs
struct HashesVisitor;
```

### 3. Deserialization Logic

```rs
impl<'de> Visitor<'de> for HashesVisitor {
    type Value = Hashes;

    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        formatter.write_str("a byte string whose length is multiple of 20")
    }

    fn visit_bytes<E>(self, v: &[u8]) -> Result<Self::Value, E>
    where
        E: de::Error,
    {
        if v.len() % 20 != 0 {
            return Err(E::custom(format!("length is {}", v.len())));
        }

        Ok(Hashes(
            v.chunks_exact(20)
                .map(|slice_20| slice_20.try_into().expect("guaranteed to be length 20"))
                .collect(),
        ))
    }
}
```

Here, we define how to convert a byte string into our `Hashes` structure. We expect the byte string's length to be a multiple of 20, as each hash is 20 bytes. The `visit_bytes` method splits this string into 20-byte chunks and collects them into our `Hashes` struct.

### 4. Implementing Deserialize and Serialize

```rs
impl<'de> Deserialize<'de> for Hashes {
    fn deserialize<D>(deserializer: D) -> Result<Hashes, D::Error>
    where
        D: Deserializer<'de>,
    {
        deserializer.deserialize_bytes(HashesVisitor)
    }
}

impl Serialize for Hashes {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let single_file = self.0.concat();
        serializer.serialize_bytes(&single_file)
    }
}
```

These implementations tell Rust how to deserialize a byte string into Hashes and serialize Hashes back into a byte string. During serialization, we simply concatenate all 20-byte arrays into a single byte string.


## Define `Keys` enum

Because of the spec of keys, we need to use [untagged](https://serde.rs/variant-attrs.html#untagged) attribute of [serde](https://docs.rs/serde/latest/serde/). 
The untagged attribute allows us to deserialize the data without explicitly specifying which variant of the enum to use, which is important here since a .torrent file will have either `length` or `files`, but not both.

> There is also a key length or a key files, but not both or neither. If length is present then the download represents a single file, otherwise it represents a set of files which go in a directory structure.


```rs
/// There is also a key `length` or a key `files`, but not both or neither.
#[derive(Debug, Clone, Deserialize, Serialize)]
#[serde(untagged)]
pub enum Keys {
    /// If `length` is present then the download represents a single file,
    SingleFile {
        /// The length of the file in bytes.
        length: usize,
    },

    /// Otherwise it represents a set of files which go in a directory structure.
    /// For the purposes of the other keys in `Info`, the multi-file case is treated as only having
    /// a single file by concatenating the files in the order they appear in the files list.
    MultiFile {
        /// The files list is the value files maps to, and is a list of dictionaries containing the
        /// following keys:
        files: Vec<File>,
    },
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct File {
    /// The length of the file, in bytes
    pub length: usize,

    /// Subdirectory names, the last of which is the actual file name
    /// (a zero length list is an error case).
    pub path: Vec<String>,
}
```

`SingleFile` is used when the torrent represents a single file, with length indicating the size of this file. Conversely, `MultiFile` is used for torrents representing multiple files, where files is a list of `File` structs, each representing an individual file in the torrent.


## Printing a torrent info

We are goint to use the function that we explored in the [last blog](https://www.nxted.co.jp/blog/blog_detail?id=40). To handle command easily, we are going to use [clap crate](https://docs.rs/clap/latest/clap/). 

```rs
// main.rs

use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(author, version, about, long_about = None)]
struct Args {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    Decode {
        value: String,
    },
}

fn main() -> anyhow::Result<()> {
  let args = Args::parse();

  match args.command {
    Commands::Decode { value } => {
      let v = decode_bencode_value(&value).0;
      println!("{v}");
    }
    Commands::Info { torrent } => {
      let t = Torrent::read(torrent).await?;

      let file_length = match t.info.keys {
        Keys::SingleFile { length } => length,
        Keys::MultiFile { ref files } => files.iter().map(|file| file.length).sum(),
      };

      println!("Tracker URL: {}", t.announce);
      println!("Length: {}", file_length);

      let info_hash = t.info_hash();
      println!("Info Hash: {}", hex::encode(info_hash));

      println!("Piece Hashes:");
      for piece in &t.info.pieces.0 {
        println!("{}", hex::encode(piece));
      }

    }

  }

  Ok(())
}

```

## Run the app

To run command easily, we prepare the script file which is `build.sh` and put in working repo. 

```script
cargo r -- "$@"
```

Open the favorite [terminal](https://alacritty.org/), we can run the command. 

```bash
./build.sh info sample.torrent

// output
Tracker URL: http://bittorrent-test-tracker.codecrafters.io/announce
Length: 92063
Info Hash: d69f91e6b2ae4c542468d1073a71d4ea13879a7f
Piece Hashes:
e876f67a2a8886e8f36b136726c30fa29703022d
6e2275e604a0766656736e81ff10b55204ad8d35
f00d937a0213df1982bc8d097227ad9e909acc17
```


## Conclusion

In this blog, we've taken a deep dive into the intricacies of parsing a .torrent file using Rust. Starting with a basic understanding of what a .torrent file is and its crucial role in the BitTorrent protocol, we explored the detailed structure of such files. We defined and implemented the Torrent and Info structs, providing clarity on how the .torrent file's various components, such as announce, info, pieces, and keys, are represented and processed.

We also tackled the more complex aspects of serialization and deserialization, particularly focusing on handling the pieces field with the Visitor pattern. This not only highlighted the flexibility and power of Rust in handling such tasks but also showcased the importance of correctly managing these data structures for the integrity of file-sharing in BitTorrent.

Furthermore, we explored the Keys enum, understanding its critical role in differentiating between single and multiple file torrents. This understanding is vital for anyone looking to develop or work with BitTorrent clients, as it directly impacts how files are downloaded and managed.

Finally, we brought everything together with a practical example, using the clap crate for command-line interaction, demonstrating how the concepts we discussed can be applied in a real-world scenario. This hands-on approach not only solidifies the theoretical aspects covered but also provides a useful guide for those looking to implement their own torrent file parsers.

Through this journey, we've seen how combining Rust's powerful features with a clear understanding of the BitTorrent file structure can lead to effective and efficient parsing solutions. Whether you're a seasoned Rust developer or just starting out, this exploration serves as a comprehensive guide to understanding and working with .torrent files.

As we continue to explore the vast and dynamic world of peer-to-peer file sharing, the knowledge and skills gained here lay a strong foundation for further exploration and innovation in this field.


## Up Next

Stay tuned for our next blog, where we will delve into the next step of interacting with a BitTorrent tracker, expanding our understanding and capabilities in the BitTorrent ecosystem.

Thank you for joining me on this deep dive into the world of BitTorrent file parsing. If you have any questions, thoughts, or insights, feel free to share them in the comments below. Happy coding!

## References
- https://news.ycombinator.com/item?id=37941075
- https://blog.jse.li/posts/torrent/
