## Introduction

In [our last exploration](https://www.nxted.co.jp/blog/blog_detail?id=49), we delved into the intricacies of communicating with a BitTorrent tracker, breaking down the request components essential for successful peer discovery. Building on that foundation, we now advance to establishing direct communication with peers. In this installment, we'll navigate the crucial step of initiating a handshake with a peer, a gateway to peer-to-peer file sharing in the BitTorrent ecosystem. Mastering the handshake process is pivotal, as it not only validates the connection between peers but also sets the stage for subsequent data exchange, laying the groundwork for efficient file sharing. Ever wondered how your BitTorrent client selects peers for file exchange or how it ensures secure and reliable connections? In this post, we demystify the handshake protocol that makes it all possible. If you're new to our series or need a refresher on the fundamentals of tracker communication, I encourage you to revisit [our previous discussion](https://www.nxted.co.jp/blog/blog_detail?id=49) for a comprehensive overview.

## Handshake

First of all, we have to define a handshake struct. 
`Handshake` struct has 5 attributes according to the [spec](https://www.bittorrent.org/beps/bep_0003.html#peer-protocol).

#### length 
length of the protocol string (BitTorrent protocol) which is 19 (1 byte)

#### protocol
the string BitTorrent protocol (19 bytes)

#### reserved
eight reserved bytes, which are all set to zero (8 bytes)

#### info_hash
sha1 infohash (20 bytes) (NOT the hexadecimal representation, which is 40 bytes long)

#### peer_id
peer id (20 bytes) (you can use 00112233445566778899 for this blog)

```rust
#[derive(Debug, Clone)]
pub struct Handshake {
    pub length: u8,
    pub protocol: Vec<u8>,
    pub reserved: Vec<u8>,
    pub info_hash: Vec<u8>,
    pub peer_id: Vec<u8>,
}
```

To initiate the new Handshake, we are going to create new function. 

```rust
impl Handshake {
    pub fn new(info_hash: &[u8; 20]) -> Self {
        Self {
            length: 19,
            protocol: b"BitTorrent protocol".to_vec(),
            reserved: vec![0; 8],
            info_hash: info_hash.to_vec(),
            peer_id: b"00112233445566778899".to_vec(),
        }
    }

   pub fn bytes(&self) -> Vec<u8> {
     let mut bytes = Vec::with_capacity(68);

     bytes.push(self.length);
     bytes.extend(self.protocol.clone());
     bytes.extend(self.reserved.clone());
     bytes.extend(self.info_hash.clone());
     bytes.extend(self.peer_id.clone());

     bytes
   }
}
```


## Command

To make life easier, we are going to do step by step. 

1. Define the handshake command

We are going to take 2 arguments `torrent`, `peer`. `torrent` is the path of torrent file and `peer` is address of the peer that we are try to connect and handshake with. 

```rust
#[derive(Subcommand)]
enum Commands {
    Handshake {
        torrent: PathBuf,
        peer: String,
    },
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
  let args = Args::parse();

  match args.command {
    Commands::Handshake { torrent, peer } => {}
  }
}

```

2. Read the torrent file

Inside handshake comamnd, we going to read the torrent file and deseriaze into actual `Torrent` struct to get the information. 

```rust
let dot_torrent = std::fs::read(torrent).context("read torrent file")?;
let t: Torrent = serde_bencode::from_bytes(&dot_torrent).context("parse torrent file")?;
```

3. Calculate info hash

In order to handshake with the peer, we have to send the info hash. 

```rust
let info_hash = t.info_hash();
```

Inside `info_hash` function, we convert the `info` to bytes, then hash them by `Sha1`. 

```rust
impl Torrent {
  pub fn info_hash(&self) -> [u8; 20] {
    let info_encoded = serde_bencode::to_bytes(&self.info).expect("re-encode info section");
    let mut hasher = Sha1::new();
    hasher.update(&info_encoded);
    hasher
      .finalize()
      .try_into()
      .expect("GenericArray<[u8; 20]>")
  }
}
```

3. Connect with the peer

From the argument of peer info, we try to connect with it. By [tokio](https://docs.rs/tokio/latest/tokio/), connect asynchronously. 

```rust
let mut res = tokio::net::TcpStream::connect(peer)
  .await;
```

Error Handling in the Handshake Process
When connecting to a peer and performing a handshake, various errors can occur, such as connection failures, I/O errors, or mismatches in the handshake response. Properly handling these errors not only makes your application more reliable but also aids in debugging and maintenance. To interact with error handling, very helpful with [anyhow](https://docs.rs/anyhow/latest/anyhow/) crate.

```rust
let mut peer_stream = match res {
    Ok(stream) => stream,
    Err(e) => {
        eprintln!("Failed to connect to peer {}: {}", peer, e);
        return Err(anyhow::Error::new(e)); // Convert the error to anyhow::Error if using anyhow for error handling
    },
};
```


4. Handshake

```rust
// Default handshake creation logic
let handshake = Handshake::new(info_hash);
{
  let mut handshake_bytes = handshake.bytes();
  stream.write_all(&mut handshake_bytes).await?;

  stream.read_exact(&mut handshake_bytes).await?;
}
```

5. Get peer id from the peer

```rust
assert_eq!(handshake.length, 19);
assert_eq!(handshake.bittorent_protocol, *b"BitTorrent protocol");

println!("Peer ID: {}", hex::encode(handshake.peer_id));
```

When we hit the command like below, we would get the peer id. 

```bash
./build.sh handshake sample.torrent 178.62.82.89:51470
```

## Conclusion
We explored discovering peers and tried to handshake with one of the peer, then we successfully got the peer id.
In the next blog, we will download the piece of file from the peer. 
Thank you for reading.

## Resources
- [The BitTorrent Protocol Specification](https://www.bittorrent.org/beps/bep_0003.html#peer-protocol)

