## Introduction

In [our last exploration](https://www.nxted.co.jp/blog/blog_detail?id=55), we delved into discovering peers and tried to handshake with one of the peer, then we successfully got the peer id.
By using peer id, in this time, we going to try exchange the message with the peer then download a piece.


## Flow

Before diving into the section of downloading a piece, we want to check the flow.
To download a piece, our program need to send [peer messages](https://www.bittorrent.org/beps/bep_0003.html#peer-messages) to a peer. The overall flow look like this:

1. Read the torrent file to get the tracker file. 

    check out [here](https://www.nxted.co.jp/blog/blog_detail?id=48)

2. Perform the tracker GET request to get a list of peers

    check out [here](https://www.nxted.co.jp/blog/blog_detail?id=49)

3. Establish TCP connection with a peer, and perform a handshake

    check out [here](https://www.nxted.co.jp/blog/blog_detail?id=55)

4. Exchange multiple peer messages to download the file

    later in this blog and next blog...


## Peer Messages

Peer messages consists of a message length prefix (4 bytes), message id (1 byte) and a payload (variable size). Here are the peer messages you'll need to exchange once the handshake is complete.

1. Wait for a `bitfield` message from the peer indicating which pieces it has

- The message id for this message type is `5`.

2. Send an `interested` message

- The message id for `interested` is `2`.
- The payload for this message is empty.

3. Wait until you receive an `unchoke` message back

- The message id for `unchoke` is `1`.
- The payload for this message is empty.

4. Break the piece into blocks of 16 kiB (16 * 1024 bytes) and send a `request` message for each block

- The message id for `request` is `6`.
- The payload for this message consists of:

    - `index`: the zero-based piece index
    - `begin`: the zero-based byte offset within the piece. This'll be 0 for the first block, 2^14 for the second block, 2*2^14 for the third block etc.
    - `length`: the length of the block in bytes. This'll be 2^14 (16 * 1024) for all blocks except the last one. The last block will contain 2^14 bytes or less, you'll need calculate this value using the piece length.

5. Wait for a `piece` message for each block you've requested

- The message id for `piece` is `7`.
- The payload for this message consists of:
    
    - `index`: the zero-based piece index
    - `begin`: the zero-based byte offset within the piece
    - `block`: the data for the piece, usually `2^14` bytes long


## Get started

We are going to implement step by step. 

### Read the torrent file to get the tracker file

In this *Understanding BitTorrent* series, we are implementing the client based on [clap](https://docs.rs/clap/latest/clap/) crate that is very useful for command line argument parser. So we are going to keep using it. 
First of all, we need to take some arguments from the user.

```rs
#[derive(Parser)]
#[command(author, version, about, long_about = None)]
struct Args {
    #[command(subcommand)]
    command: Commands,
}


#[derive(Subcommand)]
enum Commands {
    #[clap(name = "download_piece")]
    DownloadPiece {
        #[arg(short)]
        output: PathBuf,
        torrent: PathBuf,
        piece: usize,
    },
}
```

- `output` is the path where downloaded file should put on. 
- `torrent` is the also path where torrent file exist.
- `piece` is which piece should download.

After declaring the argument of command line, we are going to use these. 
Following the last blog, we are going to use [tokio](https://docs.rs/tokio/latest/tokio/) create that is very useful when want to build an application like working asyncronously.
To handle error, we use [anyhow](https://docs.rs/anyhow/latest/anyhow/) crate is also very helpful. 

```rs
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let args = Args::parse();

    match args.command {
        Commands::DownloadPiece {
            torrent,
            output,
            piece: piece_i,
        } => {

            we are going to write the main logic here...

        }
    }
```

Because we explored the some of code, I will not explain much of them here. 

To read the torrent file, take the `torrent` path, then deserialize into actual `Torrent` struct. If you want to know more, visit [here](https://www.nxted.co.jp/blog/blog_detail?id=48#DefineaInfostruct).

```rs
let dot_torrent = std::fs::read(torrent).context("read torrent file")?;
let t: Torrent =
    serde_bencode::from_bytes(&dot_torrent).context("parse torrent file")?;
```

make sure that the piece we want to download is include in torrent file.

```rs
let file_length = if let torrent::Keys::SingleFile { length } = t.info.keys {
    length
} else {
    todo!()
};
assert!(piece_i < t.info.pieces.0.len());

```

###  Perform the tracker GET request to get a list of peers

After constructing torrent struct, we will send a request to the tracker.
Again, we explored a bunch of below code, I will not explain in detail. If you want to know more, visit [here](https://www.nxted.co.jp/blog/blog_detail?id=49).

```rs
let info_hash = t.info_hash();

let request = TrackerRequest {
    peer_id: String::from("00112233445566778899"),
    port: 6881,
    uploaded: 0,
    downloaded: 0,
    left: file_length,
    compact: 1,
};
let url_params =
    serde_urlencoded::to_string(request).context("url-encode tracker parameters")?;
let tracker_url = format!(
    "{}?{}&info_hash={}",
    t.announce,
    url_params,
    &urlencode(&info_hash)
);

let response = reqwest::get(tracker_url).await.context("query tracker")?;
let response = response.bytes().await.context("fetch tracker response")?;
let tracker_info: TrackerResponse =
    serde_bencode::from_bytes(&response).context("parse tracker response")?;
```

Hopefully, we can get a response from the tracker. In the tracker [response](https://www.nxted.co.jp/blog/blog_detail?id=49#DefineTrackerResponse), we have some of peer info.

### Establish TCP connection with a peer, and perform a handshake

In real, we don't which peer has reliable connection, so we need to find it. But in this blog, to avoid complicated code, we assume `peer0` does have a reliable connection.
We touched handshake section [here](https://www.nxted.co.jp/blog/blog_detail?id=55#Handshake).

```rs
let peer = tracker_info.peers.0[0];
let mut peer = tokio::net::TcpStream::connect(peer)
    .await
    .context("connect to peer")?;
let mut handshake = Handshake::new(info_hash, *b"00112233445566778899");
{
    let handshake_bytes = handshake.as_bytes_mut();
    peer.write_all(handshake_bytes)
        .await
        .context("write handshake")?;

    peer.read_exact(handshake_bytes)
        .await
        .context("read handshake")?;
}
```

then make sure the protocol is **BitTorrent**.

```rs
assert_eq!(handshake.length, 19);
assert_eq!(handshake.bittorent_protocol, *b"BitTorrent protocol");
```

We defined the above logic inside main.rs file. In this time, we are going to move to another part.
We want to handle each connection with peer, so we should define the another `Peer` struct. 

```rs
pub struct Peer {
    addr: SocketAddrV4,
    stream: TcpStream,
    bitfield: Bitfield,
    choked: bool,
}
```

To construct a Peer, we need to handshake with it, so inside `pub async fn new()` method, we redefine the handshake logic.

```rs
impl Peer {
    pub async fn new(addr: SocketAddrV4, info_hash: &[u8; 20]) -> anyhow::Result<Self> {
        let mut stream = TcpStream::connect(addr).await.context("connect to peer")?;

        let handshake = Handshake::new(info_hash);
        {
            let mut handshake_bytes = handshake.bytes();
            stream.write_all(&mut handshake_bytes).await?;

            stream.read_exact(&mut handshake_bytes).await?;
        }

        // `anhow::ensure` is a [macro](https://docs.rs/anyhow/latest/anyhow/macro.ensure.html) similar to assert_eq!();
        anyhow::ensure!(handshake.length == 19);
        anyhow::ensure!(handshake.protocol == *b"BitTorrent protocol");

        let bitfield = Message::decode(&mut stream).await?;
        anyhow::ensure!(bitfield.id == MessageId::Bitfield);
        eprintln!("Received bitfield");

        Ok(Self {
            addr,
            stream,
            bitfield: Bitfield::from_payload(bitfield.payload),
            choked: true,
        })
    }
}
```


## Exchange multiple peer messages

After completing the initial handshake, we have the capability to exchange messages. However, this isn't entirely straightforward—if the other party isn't prepared to accept messages, we must wait until they indicate their readiness. At this point, we're in a state known as being "choked" by the other party. They will issue an "unchoke" message when they're ready to communicate, signaling that it's okay for us to start requesting data. We operate under the assumption that we are choked until notified otherwise.

Once we've received the unchoke notification, we're able to start requesting specific pieces of data, and in return, they can send us the requested pieces through messages.

### Interpreting messages

A message has a length, an ID and a payload. On the wire, it looks like:


|  length: 4  | id: 1 | optional payload .... |

Each message begins with a length indicator, a 32-bit integer comprised of four bytes in big-endian order, which specifies the message's total byte count. Following this, the ID byte identifies the message type we're dealing with; for instance, an ID of 2 signifies an "interested" message. The remainder of the message, if any, is made up of the optional payload that completes the message's specified length.

From the [spec](https://www.bittorrent.org/beps/bep_0003.html#peer-messages), we have to define the message id. 

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum MessageId {
    Choke = 0,
    Unchoke = 1,
    Interested = 2,
    NotInterested = 3,
    Have = 4,
    Bitfield = 5,
    Request = 6,
    Piece = 7,
    Cancel = 8,
    Error,
}

// [From](https://doc.rust-lang.org/stable/std/convert/trait.From.html) is very useful to convert a type to another type.
// Because the ID is 1 byte and specific number. So we can convert the type base on which number is associated with which MessageId
impl From<u8> for MessageId {
    fn from(value: u8) -> Self {
        match value {
            0 => MessageId::Choke,
            1 => MessageId::Unchoke,
            2 => MessageId::Interested,
            3 => MessageId::NotInterested,
            4 => MessageId::Have,
            5 => MessageId::Bitfield,
            6 => MessageId::Request,
            7 => MessageId::Piece,
            8 => MessageId::Cancel,
            _ => MessageId::Error,
        }
    }
}
```

To interpret message, we have to define the struct for that. We have `length`, `id` and `payload` based on the spec.

```rs
pub struct Message {
    pub length: u32,
    pub id: MessageId,
    pub payload: Vec<u8>,
}



impl Message {
    // Serialize serializes a message into a buffer of the form
    // <length prefix><message ID><payload>
    pub async fn encode<W>(w: &mut W, id: MessageId, payload: &mut [u8]) -> anyhow::Result<()>
    where
        W: AsyncWrite + Unpin,
    {
        let len_buf = (payload.len() + 1) as u32;

        w.write_u32_le(len_buf).await?;
        w.write_u8(id.into()).await?;
        w.write_all(payload).await?;
        w.flush().await?;

        Ok(())
    }
}
```

To read a message from a stream, we just follow the format of a message. We read four bytes and interpret them as a `u32` to get the length of the message. Then, we read that number of bytes to get the ID (the first byte) and the payload (the remaining bytes).

```rs
impl Message {
    pub async fn decode<R>(buf: &mut R) -> anyhow::Result<Self>
    where
        R: AsyncRead + Unpin,
    {
        let length = buf.read_u32().await.context("can not read length u32")?;
        let id = buf.read_u8().await.context("can not id length u32")?;
        let mut payload = vec![0; (length - 1) as usize];
        buf.read_exact(&mut payload).await?;

        Ok(Self {
            length,
            id: MessageId::from(id),
            payload,
        })
    }
}
```

### Bitfields

The bitfield message stands out as a particularly intriguing form of communication. It serves as a compact method for peers to denote the data pieces they can share, using a structure akin to a byte array. To determine the pieces a peer possesses, one simply examines the bit positions set to 1 within the array.

```rs
pub struct Bitfield {
    payload: Vec<u8>,
}

impl Bitfield {
    pub(crate) fn has_piece(&self, piece_i: usize) -> bool {
        let byte_i = piece_i / 8;
        let bit_i = (piece_i % 8) as u32;

        let Some(&byte) = self.payload.get(byte_i) else {
            return false;
        };

        byte & (1u8.rotate_right(bit_i + 1)) != 0
    }

    pub(crate) fn pieces(&self) -> impl Iterator<Item = usize> + '_ {
        self.payload.iter().enumerate().flat_map(|(byte_i, &byte)| {
            (0..u8::BITS).filter_map(move |bit_i| {
                let piece_i = byte_i * (u8::BITS as usize) + (bit_i as usize);
                let mask = 1_u8.rotate_right(bit_i + 1);
                (byte & mask != 0).then_some(piece_i)
            })
        })
    }

    pub(crate) fn from_payload(payload: Vec<u8>) -> Self {
        Self { payload }
    }
}
```

## Download Piece!

We need to have all the tools we need to download a piece of torrent. So we're going to implement the logic.
Beccause each peer will download a piece, we can define the method inside `impl Peer`.

```rs
const BLOCK_SIZE: u32 = 1 << 14;

impl Peer {
    pub(crate) async fn download_piece(
        &mut self,
        file_length: u32,
        npiece: u32,
        plength: u32,
    ) -> anyhow::Result<Vec<u8>> {
        println!("start downloading piece: {npiece}, piece length: {plength}");

        // Send an `interested` message
        Message::encode(&mut self.stream, MessageId::Interested, &mut []).await?;

        // Wait until you receive an `unchoke` message back
        let unchoke = Message::decode(&mut self.stream).await?;
        anyhow::ensure!(unchoke.id == MessageId::Unchoke);
        println!("Received unchoke");

        // Break the piece into blocks of 16 kiB (16 * 1024 bytes)
        let mut all_pieces: Vec<u8> = Vec::new();
        let piece_length = plength.min(file_length - plength * npiece);
        let total_blocks = if piece_length % BLOCK_SIZE == 0 {
            piece_length / BLOCK_SIZE
        } else {
            (piece_length / BLOCK_SIZE) + 1
        };

        for nblock in 0..total_blocks {
            let block_req = block::Request::new(npiece as u32, nblock, piece_length);
            let mut block_payload = block_req.encode();

            // Send a `request` message for each block
            Message::encode(&mut self.stream, MessageId::Request, &mut block_payload).await?;

            let piece = Message::decode(&mut self.stream).await?;
            let payload_len = piece.payload.len();
            let mut payload = io::Cursor::new(piece.payload);

            let block_res = block::Response::new(&mut payload, payload_len).await?;
            all_pieces.extend(block_res.block());
        }

        Ok(all_pieces)
    }
}
```

```rs
use tokio::io::{AsyncRead, AsyncReadExt};

const BLOCK_SIZE: u32 = 1 << 14;

#[derive(Debug, Clone)]
pub struct Request {
    pub piece_index: u32,
    pub begin: u32,
    pub length: u32,
}

impl Request {
    pub fn new(piece_index: u32, remaining_piece: u32, plength: u32) -> Self {
        let begin = plength - remaining_piece;
        let block_size = std::cmp::min(BLOCK_SIZE, remaining_piece);

        Self {
            piece_index,
            begin,
            length: block_size,
        }
    }

    pub fn encode(&self) -> Vec<u8> {
        let mut payload = Vec::new();

        payload.extend(u32::to_be_bytes(self.piece_index));
        payload.extend(u32::to_be_bytes(self.begin));
        payload.extend(u32::to_be_bytes(self.length));

        payload
    }
}

#[derive(Debug, Clone)]
pub struct Response {
    index: u32,
    begin: u32,
    block: Vec<u8>,
}

impl Response {
    pub async fn new<R>(buf: &mut R, payload_length: usize) -> anyhow::Result<Self>
    where
        R: AsyncRead + Unpin,
    {
        let index = buf.read_u32().await?;
        let begin = buf.read_u32().await?;

        let block_len = payload_length - 4 - 4;
        let mut block = vec![0; block_len];
        buf.read_exact(&mut block).await?;

        Ok(Self {
            index,
            begin,
            block,
        })
    }

    pub fn index(&self) -> u32 {
        self.index
    }

    pub fn begin(&self) -> u32 {
        self.begin
    }

    pub fn block(&self) -> &[u8] {
        &self.block
    }
}
```

The whole code is [here](https://github.com/aoikurokawa/bittorrent-cli/).


## Conslusion

Thank you for reading. Next blog, we try to download a whole file.
See you soon!


## Resources
- [The BitTorrent Protocol Specification](https://www.bittorrent.org/beps/bep_0003.html#peer-messages)

