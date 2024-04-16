## Introduction to BitTorrent Tracker Interaction with Rust

Welcome back to our exploration of the BitTorrent protocol! In [our previous blog](https://www.nxted.co.jp/hp/blog/blog_detail?id=49), we delved into the intricacies of parsing torrent files, uncovering valuable information like tracker_url, info_hash, piece_length, and piece_hashes. This knowledge laid the groundwork for understanding the fundamental components of BitTorrent's architecture. Today, we're taking a significant leap forward by focusing on a crucial aspect of the BitTorrent ecosystem - interacting with a tracker.

### Why Trackers Matter

In the world of BitTorrent, trackers play a pivotal role. They act as the orchestrators in the peer-to-peer network, guiding peers towards each other, thus facilitating the actual file sharing. Without this interaction, locating peers who have the files you need or to whom you can upload parts you already have would be akin to finding a needle in a haystack. The tracker's ability to efficiently manage peer connections is what makes BitTorrent an effective and widely used file-sharing protocol.

### The Technical Journey Ahead

In this post, we're going to simulate the role of a BitTorrent client. Our journey involves crafting a TrackerRequest to communicate with a tracker. We'll break down the components of this request, understanding each element's purpose and how they collectively contribute to successful peer discovery. This is not just about sending a request but about establishing a two-way communication that is fundamental to the BitTorrent file transfer process.

We'll also be decoding the TrackerResponse, which is crucial for our client to understand who and where to connect in the vast sea of peers. This step is essential in the life cycle of a BitTorrent client as it transitions from a lone entity to a connected part of a broader network of file sharing.


## Define `TrackerRequest`

In order to interact with a tracker, we need to send a request first. So, first of all, we need to define a struct `TrackerRequest`. From the [spec](https://www.bittorrent.org/beps/bep_0003.html), we specify some inforamtion like `peer_id`, `ip` address and `port`.

```rust
/// Note: the info_hash field is _not_ included.
#[derive(Debug, Clone, Serialize)]
pub struct TrackerRequest {
    /// A unique identifier for your client
    ///
    /// A string of length 20 that you get to pick.
    pub peer_id: String,

    /// The port your client is listening on
    pub port: u16,

    /// The total amount uploaded so far
    pub uploaded: usize,

    /// The total amount downloaded so far
    pub downloaded: usize,

    /// The number of bytes left to download
    pub left: usize,

    /// whether the peer list should use the compact representation
    ///
    /// The compact representation is more commonly used in the wild, the non-compact representation is mostly supported for backward-compatibility.
    pub compact: u8,
}
```

> info_hash: 
> The 20 byte sha1 hash of the bencoded form of the info value from the metainfo file. This value will almost certainly have to be escaped.
> Note that this is a substring of the metainfo file. The info-hash must be the hash of the encoded form as found in the .torrent file, which is identical to bdecoding the metainfo file, extracting the info dictionary and encoding it if and only if the bdecoder fully validated the input (e.g. key ordering, absence of leading zeros). Conversely that means clients must either reject invalid metainfo files or extract the substring directly. They must not perform a decode-encode roundtrip on invalid data.

> peer_id: 
> A string of length 20 which this downloader uses as its id. Each downloader generates its own id at random at the start of a new download. This value will also almost certainly have to be escaped.

> ip: 
> An optional parameter giving the IP (or dns name) which this peer is at. Generally used for the origin if it's on the same machine as the tracker.

> port: 
> The port number this peer is listening on. Common behavior is for a downloader to try to listen on port 6881 and if that port is taken try 6882, then 6883, etc. and give up after 6889.

> uploaded: 
> The total amount uploaded so far, encoded in base ten ascii.

> downloaded: 
> The total amount downloaded so far, encoded in base ten ascii.

> left: 
> The number of bytes this peer still has to download, encoded in base ten ascii. Note that this can't be computed from downloaded and the file length since it might be a resume, and there's a chance that some of the downloaded data failed an integrity check and had to be re-downloaded.

> event: 
> This is an optional key which maps to started, completed, or stopped (or empty, which is the same as not being present). If not present, this is one of the announcements done at regular intervals. An announcement using started is sent when a download first begins, and one using completed is sent when the download is complete. No completed is sent if the file was complete when started. Downloaders send an announcement using stopped when they cease downloading.

 We will not include `info_hash` in `TrackerRequest`. There is [one issue](https://github.com/seanmonstar/reqwest/issues/1613) that bytes can not be encoded well by [serde_urlencoded](https://github.com/nox/serde_urlencoded). So we will encode bytes manually. 

```rust
pub fn urlencode(t: &[u8; 20]) -> String {
    let mut encoded = String::new();
    for &byte in t {
        encoded.push('%');
        encoded.push_str(&hex::encode(&[byte][..]));
    }
    encoded
}
```

## Define `TrackerResponse`

Once we send a request to the tracker, the tracker will response. So we need to define the sturct of it. 

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct TrackerResponse {
    /// An integer, indicating how often your client should make a request to the tracker in
    /// seconds.
    /// You can ignore this value for the purposes of this challenge.
    pub interval: usize,

    /// A string, which contains list of peers that your client can connect to.
    ///
    /// Each peer is represented using 6 bytes.
    /// The first 4 bytes are the peer's IP address and the last 2 bytes are the peer's port number.
    pub peers: Peers,
}
```

> Tracker responses are bencoded dictionaries. If a tracker response has a key failure reason, then that maps to a human readable string which explains why the query failed, and no other keys are required. Otherwise, it must have two keys: interval, which maps to the number of seconds the downloader should wait between regular rerequests, and peers. peers maps to a list of dictionaries corresponding to peers, each of which contains the keys peer id, ip, and port, which map to the peer's self-selected ID, IP address or dns name as a string, and port number, respectively. Note that downloaders may rerequest on nonscheduled times if an event happens or they need more peers.

Each peer is represented using 6 bytes. The first 4 bytes are the peer's IP address and the last 2 bytes are the peer's port number.
To serialize and deserialize bytes, we will use same method that I introduced [before](https://www.nxted.co.jp/hp/blog/blog_detail?id=49#UnderstandingpiecesinaTorrentFile). 

```rust
    #[derive(Debug, Clone)]
    pub struct Peers(pub Vec<SocketAddrV4>);
    struct PeersVisitor;

    impl<'de> Visitor<'de> for PeersVisitor {
        type Value = Peers;

        fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
            formatter.write_str("6 bytes. the first 4 bytes are the peer's IP address and the last 2 bytes are the peer's port number.")
        }

        fn visit_bytes<E>(self, v: &[u8]) -> Result<Self::Value, E>
        where
            E: de::Error,
        {
            if v.len() % 6 != 0 {
                return Err(E::custom(format!("length is {}", v.len())));
            }

            Ok(Peers(
                v.chunks_exact(6)
                    .map(|slice_6| {
                        SocketAddrV4::new(
                            Ipv4Addr::new(slice_6[0], slice_6[1], slice_6[2], slice_6[3]),
                            u16::from_be_bytes([slice_6[4], slice_6[5]]),
                        )
                    })
                    .collect(),
            ))
        }
    }

    impl<'de> Deserialize<'de> for Peers {
        fn deserialize<D>(deserializer: D) -> Result<Peers, D::Error>
        where
            D: Deserializer<'de>,
        {
            deserializer.deserialize_bytes(PeersVisitor)
        }
    }

    impl Serialize for Peers {
        fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
        where
            S: Serializer,
        {
            let mut single_slice = Vec::with_capacity(6 * self.0.len());
            for peer in &self.0 {
                single_slice.extend(peer.ip().octets());
                single_slice.extend(peer.port().to_be_bytes());
            }
            serializer.serialize_bytes(&single_slice)
        }
    }
```

## Let's send a request

In the `main.rs`, we will make new command for discovering peers. 

```rust
#[derive(Subcommand)]
enum Commands {
    Peers {
        torrent: PathBuf,
    },
}
```

To send a request to the tracker, we will use [reqwest](https://docs.rs/reqwest/latest/reqwest/) crate. 

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let args = Args::parse();

    match args.command {
        Commands::Peers { torrent } => {
            let dot_torrent = std::fs::read(torrent).context("read torrent file")?;
            let t: Torrent =
                serde_bencode::from_bytes(&dot_torrent).context("parse torrent file")?;

            let length = if let torrent::Keys::SingleFile { length } = t.info.keys {
                length
            } else {
                todo!()
            };

            let info_hash = t.info_hash();

            let request = TrackerRequest {
                peer_id: String::from("00112233445566778899"),
                port: 6881,
                uploaded: 0,
                downloaded: 0,
                left: length,
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
            let response: TrackerResponse =
                serde_bencode::from_bytes(&response).context("parse tracker response")?;

            for peer in response.peers.0 {
                println!("{}:{}", peer.ip(), peer.port());
            }
        }
    }

    Ok(())
}
```

When we hit the command below, we will get the info of peers. 

```bash
./build.sh peers sample.torrent

# Output
# 178.62.82.89:51470
# 165.232.33.77:51467
# 178.62.85.20:51489
```

## Conclusion

As we conclude today's deep dive into BitTorrent's tracker interaction using Rust, we have journeyed through the intricate process of establishing communication with a tracker. We've seen firsthand how a `TrackerRequest` is crafted and sent, and how vital the `TrackerResponse` is in guiding our client within the peer-to-peer network.

### Looking Ahead

The next stage is we try to initiate a handshake with peers. This step is critical for actual data exchange in the BitTorrent network, and mastering it will mark a significant milestone in understanding and harnessing the full potential of BitTorrent.

Thank you for reading.

## Resources
- [The BitTorrent Protocol Specification](https://www.bittorrent.org/beps/bep_0003.html)
- [Building a BitTorrent client from the ground up in Go](https://blog.jse.li/posts/torrent/)
