# influxdb2

This is a Rust client to InfluxDB using the [2.0 API][2api].

[2api]: https://v2.docs.influxdata.com/v2.0/reference/api/

This project is a fork from the 
https://github.com/influxdata/influxdb_iox/tree/main/influxdb2_client project.
At the time of this writing, the query functionality of the influxdb2 client 
from the official repository isn't working. So, I created this client to use 
it in my project.

## Setup

Add this to `cargo.toml`:

```toml
influxdb2 = "0.3"
influxdb2-structmap = "0.2"
num-traits = "0.2"
```

## TLS Implementations
This crate uses [reqwest](https://github.com/seanmonstar/reqwest) under the hood.
You can choose between `native-tls` and `rustls` with the features provided with this crate.
`native-tls` is chosen as the default, like reqwest does.

```toml
# Usage for native-tls (the default).
influxdb2 = "0.3"

# Usage for rustls.
influxdb2 = { version = "0.3", features = ["rustls"], default-features = false }
```

## Usage

### Querying

```rust
use chrono::{DateTime, FixedOffset};
use influxdb2::{Client, FromDataPoint};
use influxdb2::models::Query;

#[derive(Debug, FromDataPoint)]
pub struct StockPrice {
    ticker: String,
    value: f64,
    time: DateTime<FixedOffset>,
}

impl Default for StockPrice {
    fn default() -> Self {
        Self {
            ticker: "".to_string(),
            value: 0_f64,
            time: chrono::MIN_DATETIME.with_timezone(&chrono::FixedOffset::east(7 * 3600)),
        }
    }
}

async fn example() -> Result<(), Box<dyn std::error::Error>> {
    let host = std::env::var("INFLUXDB_HOST").unwrap();
    let org = std::env::var("INFLUXDB_ORG").unwrap();
    let token = std::env::var("INFLUXDB_TOKEN").unwrap();
    let client = Client::new(host, org, token);

    let qs = format!("from(bucket: \"stock-prices\") 
        |> range(start: -1w)
        |> filter(fn: (r) => r.ticker == \"{}\") 
        |> last()
    ", "AAPL");
    let query = Query::new(qs.to_string());
    let res: Vec<StockPrice> = client.query::<StockPrice>(Some(query))
        .await?;
    println!("{:?}", res);

    Ok(())
}
```

### Writing

```rust
async fn example() -> Result<(), Box<dyn std::error::Error>> {
    use futures::prelude::*;
    use influxdb2::models::DataPoint;
    use influxdb2::Client;

    let host = std::env::var("INFLUXDB_HOST").unwrap();
    let org = std::env::var("INFLUXDB_ORG").unwrap();
    let token = std::env::var("INFLUXDB_TOKEN").unwrap();
    let bucket = "bucket";
    let client = Client::new(host, org, token);
    
    let points = vec![
        DataPoint::builder("cpu")
            .tag("host", "server01")
            .field("usage", 0.5)
            .build()?,
        DataPoint::builder("cpu")
            .tag("host", "server01")
            .tag("region", "us-west")
            .field("usage", 0.87)
            .build()?,
    ];
                                                            
    client.write(bucket, stream::iter(points)).await?;
    
    Ok(())
}
```

## Supported Data Types

InfluxDB data point doesn't support every data types supported by Rust. So,
the derive macro only allows for a subset of data types which is also 
supported in InfluxDB. 

Supported struct field types:

- bool
- f64
- i64
- u64 - DEPRECATED, will be removed in version 0.4
- String
- Vec<u8>
- chrono::Duration
- DateTime<FixedOffset>

## Features

Implemented API

- [x] Query API
- [x] Write API
- [x] Delete API
- [ ] Bucket API (partial: only list, create, delete)
- [ ] Organization API (partial: only list)
- [ ] Task API (partial: only list, create, delete)

## Development Status

This project is still at alpha status and all the bugs haven't been ironed 
yet. With that said, use it at your own risk and feel free to create an issue 
or pull request.

