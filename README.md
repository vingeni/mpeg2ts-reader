mpeg2ts-reader
==============

Rust reader for MPEG2 Transport Stream data

[![Build Status](https://travis-ci.org/dholroyd/mpeg2ts-reader.svg?branch=master)](https://travis-ci.org/dholroyd/mpeg2ts-reader)
[![crates.io version](https://img.shields.io/crates/v/mpeg2ts-reader.svg)](https://crates.io/crates/mpeg2ts-reader)

# Example

Dump timestamps attached to any ADTS audio or H264 video streams.

```rust
extern crate mpeg2ts_reader;

use std::env;
use std::fs::File;
use std::io::Read;
use std::collections::HashMap;
use mpeg2ts_reader::demultiplex;
use mpeg2ts_reader::pes;
use mpeg2ts_reader::StreamType;


// Implement the ElementaryStreamConsumer to just dump and PTS/DTS timestamps to stdout
struct PtsDumpElementaryStreamConsumer {
    pid: u16,
}
impl PtsDumpElementaryStreamConsumer {
    fn construct(_pmt_sect: &demultiplex::PmtSection, stream_info: &demultiplex::StreamInfo)
        -> Box<std::cell::RefCell<demultiplex::PacketFilter>>
    {
        let consumer = pes::PesPacketConsumer::new(
            PtsDumpElementaryStreamConsumer {
                pid: stream_info.elementary_pid(),
            }
        );
        Box::new(std::cell::RefCell::new(consumer))
    }
}
impl pes::ElementaryStreamConsumer for PtsDumpElementaryStreamConsumer {
    fn start_stream(&mut self) { }
    fn begin_packet(&mut self, header: pes::PesHeader) {
        match header.contents() {
            pes::PesContents::Parsed(Some(parsed)) => {
                match parsed.pts_dts() {
                    pes::PtsDts::PtsOnly(Ok(pts)) => {
                        println!("PID {}: pts {:#x}", self.pid, pts.value())
                    },
                    pes::PtsDts::Both{pts:Ok(pts), dts:Ok(dts)} => {
                        println!("PID {}: pts={:#x} dts={:#x}", self.pid, pts.value(), dts.value())
                    },
                    _ => (),
                }
            },
            pes::PesContents::Parsed(None) => (),
            pes::PesContents::Payload(_) => { },
        }
    }
    fn continue_packet(&mut self, _data: &[u8]) { }
    fn end_packet(&mut self) { }
    fn continuity_error(&mut self) { }
}

fn main() {
    // open input file named on command line,
    let name = env::args().nth(1).unwrap();
    let mut f = File::open(&name).expect(&format!("file not found: {}", &name));

    // configure the stream types we will handle,
    let mut table: HashMap<StreamType, fn(&demultiplex::PmtSection,&demultiplex::StreamInfo)->Box<std::cell::RefCell<demultiplex::PacketFilter>>>
        = HashMap::new();
    table.insert(StreamType::H264, PtsDumpElementaryStreamConsumer::construct);
    table.insert(StreamType::Adts, PtsDumpElementaryStreamConsumer::construct);
    let stream_constructor = demultiplex::StreamConstructor::new(
        demultiplex::NullPacketFilter::construct,
        table
    );

    // create the demultiplexer with the above config
    let mut demultiplex = demultiplex::Demultiplex::new(stream_constructor);

    // consume the input file,
    let mut buf = [0u8; 188*1024];
    loop {
        match f.read(&mut buf[..]).expect("read failed") {
            0 => break,
            n => demultiplex.push(&buf[0..n]),
        }
    }
}
```

# Performance

On my _i7-7820HQ_ laptop, a microbenchmark that parses TS structure, but ignores the audio and video contained within,
can process at a rate of **6 GiBytes/s** (48 Gibits/s).  This is when the data is already in memory (no network/disk
access), and processing is happening on a single core (no multiprocessing of the stream).

# Supported Transport Stream features

Not all Transport Stream features are supported yet.  Here's a summary of what's available,
and what's yet to come:

- Framing
  - ☑ _ISO/IEC 13818-1_ 188-byte packets
  - ☐ m2ts 192-byte packets (would be nice if an external crate could support, at least)
  - ☐ recovery after loss of synchronisation
- Transport Stream packet
  - ☑ Fixed headers
  - ☐ Adaptation field (Adaptation field size is accounted for in finding packet payload, adaptation field details are not yet exposed)
- Program Specific Information tables
  - ☑ Section syntax
  - ☐ 'Multi-section' tables
  - ☑ PAT - Program Association Table
  - ☑ PMT - Program Mapping Table
  - ☐ TSDT - Transport Stream Description Table
- Packetised Elementary Stream syntax
  - ☑ PES_packet_data
  - ☑ PTS/DTS
  - ☐ ESCR
  - ☐ ES_rate
  - ☐ DSM_trick_mode
  - ☐ additional_copy_info
  - ☐ PES_CRC
  - ☐ PES_extension
- Descriptors
  - ☐ video_stream_descriptor
  - ☐ audio_stream_descriptor
  - ☐ hierarchy_descriptor
  - ☐ registration_descriptor
  - ☐ data_stream_alignment_descriptor
  - ☐ target_background_grid_descriptor
  - ☐ video_window_descriptor
  - ☐ ca_descriptor
  - ☐ iso_639_language_descriptor
  - ☐ system_clock_descriptor
  - ☐ multiplex_buffer_utilization_descriptor
  - ☐ copyright_descriptor
  - ☐ maximum_bitrate_descriptor
  - ☐ private_data_indicator_descriptor
  - ☐ smoothing_buffer_descriptor
  - ☐ std_descriptor
  - ☐ ibp_descriptor
  - ☐ mpeg4_video_descriptor
  - ☐ mpeg4_audio_descriptor
  - ☐ iod_descriptor
  - ☐ sl_descriptor
  - ☐ fmc_descriptor
  - ☐ external_es_id_descriptor
  - ☐ muxcode_descriptor
  - ☐ fmxbuffersize_descriptor
  - ☐ multiplexbuffer_descriptor
