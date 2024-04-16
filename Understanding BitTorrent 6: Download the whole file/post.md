## Introduction

In the vast landscape of modern software development, the efficiency and security of data transfer across networks are paramount, particularly when dealing with large files. BitTorrent, a peer-to-peer (P2P) file-sharing protocol, has been a popular choice for such tasks due to its robustness and decentralized nature. Unlike traditional file download methods that rely on a single server, BitTorrent distributes the load across multiple points, reducing the risk of bottlenecks and increasing download speeds.

However, implementing a BitTorrent client comes with its own set of challenges, including handling concurrency, managing file transfers, and ensuring data integrity. This is where Rust, with its focus on performance and safety, shines. Rust’s powerful concurrency features and strong compile-time guarantees make it an ideal choice for building reliable network applications.

In this blog post, we will dive deep into the creation of a basic BitTorrent client using Rust. We'll start by setting up our project, parsing command-line arguments, and managing asynchronous file downloads. Whether you're a seasoned Rustacean or new to the language, this guide will provide you with a practical understanding of how to leverage Rust's capabilities for network programming. By the end of this tutorial, you'll have a working BitTorrent client that can download files from peers, and you'll gain insights into Rust’s asynchronous programming model that could be applied to a wide range of other projects.

If you are curious about downloading a piece, you can read [here](https://www.nxted.co.jp/hp/blog/blog_detail?id=62).

If you want to read a whole series, I recommend reading from [here](https://www.nxted.co.jp/blog/blog_detail?id=40).

## Overview of the Code 

Our BitTorrent client in Rust is structured around several key components, each playing a critical role in the functionality of the application. The program is designed to handle command-line arguments, read torrent file metadata, manage downloads, and save files to the disk. Let’s break down the primary sections of our code to better understand their purposes and interactions.

### Command Line Parsing

The backbone of user interaction with our BitTorrent client is the clap crate, which we use to parse command-line arguments. Our implementation utilizes the Args struct to define expected inputs and the Commands enum to differentiate between various user commands, focusing on the Download command for this tutorial. This approach allows us to neatly encapsulate command options and provide a clear interface for users to interact with the program.

### Asynchronous Setup

Given the I/O-heavy nature of a file download application, asynchronous programming is essential. We leverage Rust’s tokio runtime to handle asynchronous tasks efficiently. The use of the #[tokio::main] macro simplifies the setup of the async environment, allowing us to focus on the logic of handling torrent file operations and network I/O without worrying about the underlying thread management.
Reading Torrent Files

The initial step in any BitTorrent download is to parse the .torrent file, which contains metadata about the files to be downloaded. Our Torrent::read function is designed to asynchronously read and parse this metadata, preparing the client to manage the download process. This functionality is crucial for understanding what needs to be downloaded and how the pieces of the file are structured across different peers.

### Download Management

Once the torrent metadata is loaded, the core of our application involves managing the downloads. This includes connecting to peers, requesting pieces of the file, and handling the data as it is received. Our download logic checks the type of file structure (single or multiple files) and processes the incoming data accordingly, ensuring that each piece is correctly assembled and saved to the local filesystem.

### File Handling

Handling files efficiently is vital in a BitTorrent client. Depending on whether the torrent describes a single file or multiple files, our application either writes a single continuous stream to the disk or handles multiple file paths and writes data to each file appropriately. This part of the code is especially sensitive to errors, so careful management of file paths and write operations is essential to avoid data corruption.

## Detailed Code Breakdown

1. Parsing Command Line Arguments

In this series, we have used [clap crate](https://docs.rs/clap/latest/clap/) for command line stuff. So we continue using it.
`Download` takes two fields.

- `output`: the path that the downloaded file should be put
- `torrent`: the path where torrent file put


```rs
#[derive(Parser)]
#[command(author, version, about, long_about = None)]
struct Args {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    Download {
        #[arg(short)]
        output: PathBuf,
        torrent: PathBuf,
    },
}
```

2. Asynchronous Main Function

[Tokio](https://docs.rs/tokio/latest/tokio/) is one of very popular crates for writing asynchronous applications with the Rust programming language.
Tokio provides [`main` macro](https://docs.rs/tokio/latest/tokio/attr.main.html) that set up a Runtime without requiring the user to use [Runtime](https://docs.rs/tokio/latest/tokio/runtime/struct.Runtime.html) or [Builder](https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html) directly.


```rs
#[tokio::main]
async fn main() -> anyhow::Result<()> {

    Ok(())
}
```

Inside main function, we have to parse the command.

```rs
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let args = Args::parse();

    match args.command {
        Commands::Download { output, torrent } => {
        }
    }

    Ok(())
}
```

3. Reading and Parsing the Torrent File

I explained about reading torrent file in previous [blog](https://www.nxted.co.jp/blog/blog_detail?id=48).  

```rs
let t = Torrent::read(torrent).await?;

println!("Starting download for {}", t.info.name);

```

4. Downloading Files

### main.rs

This is the main part of this blog.
We need to create `download::all` function which returns the file.

```rs
let files = download::all(&t).await?;

match &t.info.keys {
    Keys::SingleFile { .. } => {
        tokio::fs::write(
            &output,
            files.into_iter().next().expect("always one file").bytes(),
        )
        .await?;
    }
    Keys::MultiFile { .. } => {
        while let Some(file) = files.into_iter().next() {
            let file_path = file.path().join(std::path::MAIN_SEPARATOR_STR);
            tokio::fs::write(&file_path, file.bytes()).await?;
        }
    }
}

println!("Downloaded test.torrent to {}.", output.display());
```

### download.rs

`all` function takes a parameter `Torrent` struct which contains the information about torrent file.

```rs
pub async fn all(t: &Torrent) -> anyhow::Result<Downloaded> {

}
```

First of all, we need to discover peers that hold a piece of file.
In terms of discovering peers, I explained in this [blog](https://www.nxted.co.jp/blog/blog_detail?id=49).

```rs
pub async fn all(t: &Torrent) -> anyhow::Result<Downloaded> {
    let info_hash = t.info_hash();
    let request = tracker::http::Request::new(&info_hash, t.length());
    let addr = tracker::get_addr(&t.announce)?;

    let peers = {
        let res = reqwest::get(request.url(&url.to_string())).await?;
        let res: tracker::http::Response =
            serde_bencode::from_bytes(&res.bytes().await?).context("parse response")?;

        res.peers.0
    };
}

```

`peers` are struct of [Peers](https://github.com/aoikurokawa/bittorrent-cli/blob/8eef03ef1f34bea602766f2c8c039833695e74c2/src/tracker/http.rs#L83) that hold Vec of address.
To establish connection with peer asynchronously, we need to construct a `Peer` by calling [`Peer::new()`](https://github.com/aoikurokawa/bittorrent-cli/blob/8eef03ef1f34bea602766f2c8c039833695e74c2/src/peer.rs#L28:L52).
When dealing with I/O-bound tasks in Rust, we need to convert a regular iterator into a stream using [`futures_util::stream::iter()`](https://docs.rs/futures-util/latest/futures_util/stream/fn.iter.html) and then apply transformations such as `map` and `buffer_unordered`.
The `buffer_unordered` combinator is used to control how many asynchronous operations can run concurrently. In this case, up to 5 operations can be processed at the same time.

```rs
let mut peers = futures_util::stream::iter(peers)
    .map(|peer_addr| async move {
        let peer = Peer::new(peer_addr, &info_hash).await;
        (peer_addr, peer)
    })
    .buffer_unordered(5);

let mut peer_list = Vec::new();
while let Some((peer_addr, peer)) = peers.next().await {
    match peer {
        Ok(peer) => {
            eprintln!("Completed handshake with {peer_addr}");

            peer_list.push(peer);

            if peer_list.len() > 5 {
                break;
            }
        }
        Err(e) => {
            eprintln!("Could not handshake with {peer_addr}. Disconnecting: {e}");
        }
    }
}
drop(peers);
```

We assume all peers have piece of file. If not, fail to assert `assert!(no_peers.is_empty());`.

```rs
let mut peers = peer_list;
let mut need_pieces = BinaryHeap::new();
let mut no_peers = Vec::new();

for piece_i in 0..t.info.pieces.0.len() {
    let piece = Piece::new(piece_i, &t, &peers);
    if piece.peers().is_empty() {
        no_peers.push(piece);
    } else {
        need_pieces.push(piece);
    }
}

assert!(no_peers.is_empty());
```

Initializes a vector all_pieces filled with zeros. Its size is based on the total length of the data to be downloaded, as specified by t.length().

```rs
let mut all_pieces = vec![0; t.length()];
```

Continuously processes each piece needed for the file. The loop extracts pieces from need_pieces until none are left.

```rs
while let Some(piece) = need_pieces.pop() {

}
```

Calculates the actual length of the current piece and determines the number of blocks within that piece. This accounts for potentially variable piece sizes, especially for the last piece of the file.

```rs
let plength = piece.length();
let npiece = piece.index();
let piece_length = plength.min(t.length() - plength * npiece);
let total_blocks = if piece_length % BLOCK_SIZE as usize == 0 {
    piece_length / BLOCK_SIZE as usize
} else {
    (piece_length / BLOCK_SIZE as usize) + 1
};
```

Filters and collects peers that have the piece, ensuring that only relevant peers are used for downloading the current piece.

```rs
let peers: Vec<_> = peers
    .iter_mut()
    .enumerate()
    .filter_map(|(peer_i, peer)| piece.peers().contains(&peer_i).then_some(peer))
    .collect();

```

Initializes channels for managing block download tasks. submit is used to dispatch block download requests, and finish is used to receive completed blocks.

```rs
let (submit, tasks) = kanal::bounded_async(total_blocks);
for block in 0..total_blocks {
    submit
        .send(block)
        .await
        .expect("bound holds all these limits");
}
let (finish, mut done) = tokio::sync::mpsc::channel(total_blocks);
```

Each peer is tasked asynchronously with downloading parts of the piece. FuturesUnordered allows for handling these tasks concurrently, not sequentially.

```rs
    let mut participants = futures_util::stream::FuturesUnordered::new();
    for peer in peers {
        participants.push(peer.participate(
            piece.index() as u32,
            total_blocks as u32,
            piece_length as u32,
            submit.clone(),
            tasks.clone(),
            finish.clone(),
        ));
    }
```

A loop that concurrently handles participants finishing and blocks being received. Blocks are inserted into all_blocks at the correct positions, and the loop breaks when all blocks are received or no more data is forthcoming.

```rs
let mut all_blocks: Vec<u8> = vec![0; piece_length];
let mut bytes_received = 0;
loop {
    tokio::select! {
        joined = participants.next(), if !participants.is_empty() => {
            // if a participant ends early, it's either slow or failed.
            match joined {
                None => {},
                Some(Ok(_)) => {},
                Some(Err(_)) => {},
            }
        },

        piece = done.recv() => {
        // keep track of the bytes in message
            if let Some(piece) = piece {
                // let piece = Piece::ref_from_bytes(&piece.block()[..]).expect("always get all Piece response fields from peer");
                all_blocks[piece.begin() as usize ..][..piece.block().len()].copy_from_slice(piece.block());
                bytes_received += piece.block().len();
                if bytes_received ==  piece_length {
                    break;
                }
            } else {
                break;
            }

        },
    }
}

```

After all blocks for a piece are received, their integrity is verified against the expected SHA1 hash provided in the torrent metadata.

```rs
let mut hasher = Sha1::new();
hasher.update(&all_blocks);
let hash: [u8; 20] = hasher.finalize().try_into().expect("");
assert_eq!(hash, piece.hash());
```

The fully assembled and verified piece is copied into the correct position in all_pieces, ensuring that each piece is stored in order based on its index.

```rs
all_pieces[piece.index() * t.info.plength..][..piece_length].copy_from_slice(&all_blocks);
```

Return downloded file.

```rs
Ok(Downloaded {
    bytes: all_pieces,
    files: match &t.info.keys {
        Keys::SingleFile { length } => vec![File {
            length: *length,
            path: vec![t.info.name.clone()],
        }],
        Keys::MultiFile { files } => files.clone(),
    },
})
```

It's a bit long so you might be get lost. 
I published the whole code [here](https://github.com/aoikurokawa/bittorrent-cli/tree/main).

## Conclusion
Today, we've explored the depths of creating a BitTorrent client in Rust, tackling the complexities of peer-to-peer file downloads. By breaking down the intricate details of managing asynchronous downloads, selecting peers, and handling data blocks, we've demonstrated not only Rust's capability to handle complex network operations but also its prowess in maintaining robust and efficient code execution.


Thank you for reading whole **Understanding BitTorrent** series. I hope you enjoyed it.

## Resources
- https://without.boats/
- [Async: What is blocking?](https://ryhl.io/blog/async-what-is-blocking/)
- [Green Threads in Rust](https://stanford-cs242.github.io/f17/assets/projects/2017/kedero.pdf)
