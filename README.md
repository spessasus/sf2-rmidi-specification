# Official SF2 RMIDI Specification
Original format idea by Zoltán Bacskó of [Falcosoft](https://falcosoft.hu), further expanded by spessasus.
Specification written by spessasus with the help of Zoltán.

Revision 1.19
## Preamble

<p align="justify">
MIDI files have long-faced a significant challenge: <strong>different sounds on different devices.</strong>
SF2 + MIDI combinations address this issue partially
by ensuring that playing both files through an SF2-compliant synth results in the same sound being produced. 
The RMIDI format is not new; it was originally developed by Microsoft as a RIFF wrapper for MIDI files
and later expanded by the MIDI Manufacturers Association to support embedding DLS sound banks.
However, DLS is not widely used today, whereas the SoundFont2 (SF2) format serves a similar purpose and remains quite popular.
The SF2 RMIDI format integrates MIDI and SF2 files into a single file, augmented with additional metadata. 
This document serves as the official specification for this format.
This version of RMIDI was created by Zoltán Bacskó of <a href="https://falcosoft.hu">Falcosoft</a>
and implemented in <a href="https://falcosoft.hu/softwares.html#midiplayer">Falcosoft SoundFont Midi Player 6.</a> 
I am in contact with Zoltán,
who <a href="https://www.vogons.org/viewtopic.php?p=1282710#p1282710">granted permission</a> to use this as the official specification. 

If you find any part of this specification unclear, please reach out via <a href="https://www.vogons.org/viewtopic.php?p=1281224">this thread</a> or file a <a href="https://github.com/spessasus/sf2-rmidi-specification/issues/new">GitHub issue</a> in this repository.
Also feel free to report any issues such as typos or expansions to this standard!
</p>

## Table of Contents
<!-- TOC -->
* [Official SF2 RMIDI Specification](#official-sf2-rmidi-specification)
  * [Preamble](#preamble)
  * [Table of Contents](#table-of-contents)
  * [Terminology](#terminology)
  * [Extension](#extension)
  * [RIFF Chunk](#riff-chunk)
    * [Example chunk](#example-chunk)
* [SF2 RMIDI File Specification](#sf2-rmidi-file-specification)
  * [File Structure](#file-structure)
    * [Example file](#example-file)
    * [Handling Differences](#handling-differences)
  * [INFO Chunk](#info-chunk)
    * [Metadata Chunks](#metadata-chunks)
    * [Chunk Rules](#chunk-rules)
      * [IENC Chunk Requirements](#ienc-chunk-requirements)
      * [IPIC Chunk Requirements](#ipic-chunk-requirements)
      * [DBNK Chunk](#dbnk-chunk)
  * [Embedded sound bank](#embedded-sound-bank)
    * [Bank Offset](#bank-offset)
  * [Player pseudo code](#player-pseudo-code)
  * [Software Requirements](#software-requirements)
    * [Level 1](#level-1)
    * [Level 2](#level-2)
    * [Level 3](#level-3)
    * [Level 4](#level-4)
  * [Types of RMIDI Files](#types-of-rmidi-files)
    * [Self-Contained File](#self-contained-file)
    * [External File](#external-file)
  * [Recommendations for Writing RMIDI Files](#recommendations-for-writing-rmidi-files)
  * [Example Files](#example-files)
  * [Reference Implementation](#reference-implementation)
<!-- TOC -->

## Terminology
This specification assumes familiarity with the SoundFont2 format and the Standard MIDI File (SMF) format. 
Additional terminology used in this specification includes:

- **The software**: Refers to software compliant with this specification.
- **Bit**: The most basic data structure element, either 0 or 1.
- **Byte**: A data structure element of eight bits, with no defined meaning to those bits.
- **SoundFont**: A [SoundFont2](https://musescore.org/sites/musescore.org/files/2023-01/sfspec24.pdf) compliant binary.
- **DLS**: DownLoadable Sounds. Sound bank format similar to SoundFont. Used in the older RMIDI files, not compliant with this specification.
- **Embedded sound bank:** The sound bank embedded within an RMIDI file that is used for playing back the sequence.
- **Main SoundFont:** The regular SoundFont bank loaded by the software before loading the RMIDI file.
- **Bank**: MIDI controller 0 `Bank Select MSB` and the bank number of a SoundFont preset. It's a 7-bit value, except for the SoundFont's drum presets, which use bank number 128.
- **Bank offset:** The number which gets added to each wBank field in all presets within the embedded bank.
- **RIFF**: Resource Interchange File Format. A file container format for storing data in tagged chunks.
- **Chunk**: The top-level division of a RIFF file.
- **Little Endian**: Byte ordering in memory with the least significant byte at the lowest address.
- **MIDI:** Musical Instrument Digital Interface. a technical standard that describes a communication protocol for a wide variety of electronic musical devices.
- **MIDI file/SMF:** Standard MIDI File. A sequence of MIDI messages, usually a song.
- **Note On:** A MIDI message indicating that a given note should be pressed.
- **Note Off:** A MIDI message indicating that a given note should be released.
- **GM**: General MIDI system, ignoring all Bank select messages.
- **XG**: Yamaha Extended General MIDI, an extension to the General MIDI standard created by Yamaha.
- **GS**: Roland General Standard, an extension to the General MIDI standard created by Roland.
- **Encoding**: Assigning numbers to graphical characters.
- **ASCII**: American Standard Code for Information Interchange, a character encoding standard for electronic communication.

## Extension
The file extension is `.rmi`, and the MIME type is `audio/rmid`.
Optionally the extension might be `.sfmi` to help distinguish between the older RMIDI format.
The file type should be referred to as `MIDI with embedded SF2`, `Embedded MIDI` or `SF2 RMIDI`.

## RIFF Chunk
The RMIDI format uses RIFF chunks to structure the data.

Each RIFF chunk in an RMIDI file follows this format:
- Four bytes: Chunk header in ASCII (e.g., `RIFF`)
- Four bytes: Chunk size as a 32-bit unsigned little-endian number
- Chunk data: Optionally, the first 4 bytes of the data represent the chunk type in ASCII (e.g., `sfbk`)

> **NOTE:**
> The chunk size **must be even.** 
> If the initial chunk data is odd, a padding byte of 0 must be added at the end. 
> The chunk's length **does not** include this padding byte.

> **IMPORTANT:** 
> This constraint applies only to RIFF chunks within the RMIDI file and does not affect RIFF chunks *within* the soundfont chunk.

### Example chunk
`52 49 46 46 05 00 00 00 48 65 6C 6C 6F 00`

- `52 49 46 46` - ASCII string "RIFF"
- `05 00 00 00` - 32-bit chunk length: 5
- `48 65 6C 6C 6F` - the chunk's data: ASCII string "Hello"
- `00` - a pad byte of 0 to make the total byte count even.

# SF2 RMIDI File Specification
## File Structure
An RMIDI file consists of:
- `RIFF` chunk (main chunk)
  - `RMID` ASCII string
  - `data` chunk containing the complete MIDI file (MThd, MTrk, etc.)
  - Optional `LIST` chunk: Metadata for the file, similar to SF2's chunk
    - `INFO` ASCII string
    - [Inner chunks described here](#info-chunk)
  - `RIFF` chunk: Complete soundfont binary.
    The first four bytes of this chunk should be `sfbk`, indicating a soundfont2 binary. 
  [SoundFont3 format](https://github.com/FluidSynth/fluidsynth/wiki/SoundFont3Format) is allowed.
  
### Example file
- `RIFF` chunk
  - `RMID` ASCII string
  - `data` - The MIDI file data: MThd, MTrk etc...
  - `LIST`
    - `INFO` ASCII string
    - `INAM` chunk
      - `Never Gonna Give You Up` UTF-8 string
    - `IART` chunk
      - `Rick Astley` UTF-8 string
    - `ICRD` chunk
      - `1987` UTF-8 string
    - `IENC` chunk
      - `utf-8` ASCII string
    - `DBNK` chunk
      - 16-bit integer: 1
  - `RIFF` chunk - the SoundFont binary: `sfbk`, `LIST`, `sdta` etc...

The following file structure shows that:
1. The bank offset is 1.
2. Info chunks are encoded using `UTF-8` encoding.
3. The song's title is "Never Gonna Give You Up."
4. The song's artist is "Rick Astley."
5. The song's creation date is "1987."
6. The song has an embedded sound bank.

### Handling Differences
When the file structure deviates from the above:
1. Any additional chunks after the specified ones should be ignored and preserved as-is.
2. If the chunk order differs from this specification, the file should be rejected.
3. If no soundfont bank is present, the file should use the main soundfont and assume a bank offset of 0, ignoring the DBNK chunk.
4. If the soundfont bank uses the older DLS format, software not capable of reading DLS should reject the file. 

Software that supports DLS should use the contained DLS
and assume a bank offset of **1** or try to detect the bank offset
since the older format does not specify the `DBNK` chunk.

The last two rules ensure backwards compatibility with the older RMIDI format.

## INFO Chunk
The INFO chunk describes file metadata and the soundfont's bank offset.

The INFO chunk may contain the following optional chunks:
- `DBNK` chunk: Soundfont's bank offset. See [DBNK Chunk](#dbnk-chunk) for details.
- `IENC` chunk: Encoding used for the metadata chunks: name of the encoding stored as string.
  Not case-sensitive, but lowercase is preferred (e.g., `utf-8`).
  [Software capable of reading the IENC chunk must support the following encodings](#ienc-chunk-requirements).
  Note that this field must use basic `ASCII` encoding.
- `MENC` chunk: Encoding hint for the text evens within the MIDI file. The same string format as `IENC`.
- [Metadata chunks](#metadata-chunks)

### Metadata Chunks
Below are the defined chunks containing additional information about the song:
- `INAM` chunk: Song name/title. String of any length.
- `ICOP` chunk: Copyright. String of any length.
- `IART` chunk: Artist (MIDI creator). String of any length.
- `ICRD` chunk: Creation date. String of any length.
- `IPRD` or `IALB` chunk: Album name. String of any length. It can be used interchangeably. If both exist in the file, the software should use `IALB`.
- `IPIC` chunk: Attached picture (e.g., album cover). Binary picture data. PNG or JPEG recommended.
- `IGNR` chunk: Song genre. String of any length.
- `ICMT` chunk: Comment/description. String of any length.
- `IENG` chunk: Engineer (soundfont creator). String of any length.
- `ISFT` chunk: Software used to create the file. String of any length.

### Chunk Rules
The following rules apply to the INFO chunk:
1. The order of chunks within the INFO chunk is arbitrary.
2. Chunks of length 0 are illegal and should be discarded.
3. Unknown INFO chunks should be ignored and preserved as-is.
4. If the `IENC` chunk is not specified, the software can use any encoding, but assuming `utf-8` is recommended.
5. If the `MENC` chunk is not specified, the software decides MIDI's encoding.
6. If the software can display the song's name, it should use the INAM chunk if present, ignoring the MIDI track name.
7. Compatible software may ignore all INFO chunks **except the DBNK chunk** for the most basic [level of compatibility](#level-1).
8. The chunk size must be even, as specified in the general RIFF structure.
9. The INFO chunk is optional. The software must not assume that the INFO chunk exists.

#### IENC Chunk Requirements
For Level 3 compatibility, software must support the following encodings (both lowercase and uppercase):
- `utf-8`
- `shift-jis` or `Shift_JIS` (equivalent encodings)
- `windows-1250` (Central Europe)
- `windows-1251` (Cyrillic)
- `windows-1252` (Western)
- `windows-1253` (Greek)
- `windows-1254` (Turkish)
- `windows-1255` (Hebrew)
- `windows-1256` (Arabic)
- `windows-1257` (Baltic)
- `windows-1258` (Vietnamese)

Software may decode other encodings but is not required to.

#### IPIC Chunk Requirements
For Level 4 compatibility, software must support the following image formats:
- Portable Network Graphics (PNG)
- Joint Photographic Experts Group (JPEG)

Other formats (e.g., `gif`, `webp`, `ico`) may also be supported but are not required.

#### DBNK Chunk
The DBNK chunk is an optional RIFF chunk within the RMIDI INFO List.

It describes the **bank offset** for the embedded sound bank.

It always has a length of two bytes,
with these bytes forming a **16-bit, unsigned, little-endian** number. 
If the chunk's length is not two bytes or the number is out of range, the file should be rejected.

Current boundaries are: **minimum: 0** and **maximum: 127**. 
The other byte is reserved for future use. 

If no DBNK is specified, an offset of **1** is assumed by default.
If the file does not contain any Sound bank (SF2 or DLS), the offset shall default to 0.

For general use, a bank offset of 0 is recommended as it allows bundling the soundfont and the MIDI without modification.

## Embedded sound bank
The RMI file may come with an embedded SF2 or DLS SoundFont, usually after the INFO chunk.
This sound bank provides the exclusive sounds used within the MIDI sequence,
temporarily replacing given MIDI program and bank numbers with the presets contained within the sound bank.

### Bank Offset
The bank offset adjusts every bank in the embedded sound bank
**except for bank 128** by adding itself to every patch's `wBank` field.

For example,
- If a preset named `Acoustic Piano 2` with program 0 and bank 1 exists within an RMIDI file which uses bank offset of 1,
it should effectively be interpreted as program 0 and bank 2.

- If a preset named `Standard Drum Kit` exists within the same RMIDI file with program 0 and bank 128,
the bank will remain 128.

If the resulting bank number exceeds 127 (except for drum kits) or is smaller than 0, then it should be turned into 0.


## Player pseudo code
Below is a simple JavaScript-like code for a Level 1 RMIDI-compatible player.

Note: this code does not perform any checks and assumes that the file is valid and contains all three chunks,
for the sake of simplicity.
```js
const file = open("song.rmi");
// read RIFF
const chunk = readRIFF(file);
// skip 'RMID' string
chunk.data.seek(chunk.data.position + 4);
// read 'data' chunk
const midiChunk = readRIFF(chunk.data);
const midiFile = midiChunk.data;

// read the 'LIST' INFO chunk
const info = readRIFF(chunk.data);
// skip the 'INFO' string
const infoString = info.data.seek(info.data.position + 4);
const infoList = readLIST(info.data);

// bank offset is 1 by default
let bankOffset = 1;
// if DBNK exists
if(infoList.find(infoChunk => infoChunk.header === "DBNK")) {
    // DBNK is 2 bytes signed int 16
    bankOffset = infoList["DBNK"].toSignedInt16();
}

// clamp the bank offset
bankOffset = Math.min(Math.max(0, bankOffset), 127);

// read the sound bank (not as a riff chunk but copy the binary content)
const soundFont = chunk.slice(chunk.data.position, chunk.data.length - chunk.data.position);

// initialize the synthesizer
const player = new Player(soundFont);

// adjust bank offset
for(const preset of player.soundFont.presets)
{
    preset.bankNumber += bankOffset;
}

// play the song
player.play(midiFile);
```

## Software Requirements
Not all chunks in the file must be read for the file to play correctly. Software compatibility with the RMIDI format is categorized into levels:

### Level 1
Minimum requirements for the software to be compliant. The software must:
- Read and interpret the `RMID` ASCII string as the file indicator.
- Handle the `data` chunk containing the MIDI data.
- Process the `DBNK` chunk within the INFO chunk and correctly offset the soundfont (or a bank select messages in the MIDI) based on this value.
- Read the `RIFF` chunk with the soundfont data.

### Level 2
This level requires basic interpretation of the `INFO` chunk. The software must:
- Read all Level 1 chunks.
- Interpret all metadata chunks (`INAM`, `IPRD`, `ICRD`, `ICOP`, etc.) as `ASCII` or `utf-8`.

### Level 3
This level requires support for the `IENC` chunk. The software must:
- Read all Level 1 and Level 2 chunks.
- Interpret the `IENC` chunk and support the [required encodings](#ienc-chunk-requirements).

As of 2024-08-07, Falcosoft Midi Player meets this level of compatibility.

### Level 4
This level requires support for the `IPIC` chunk. The software must:
- Read all Level 1, Level 2, and Level 3 chunks.
- Interpret the `IPIC` chunk and support the [required image formats](#ipic-chunk-requirements).

As of 2024-08-06, SpessaSynth meets this level of compatibility.

As of 2024-08-20, foo_midi meets this level of compatibility.

## Types of RMIDI Files
There are currently two distinct types of RMIDI files that vary in their use cases.

Note that these have identical file structure; these vary only in the way they provide sounds for the sequence.

### Self-Contained File
A self-contained file is defined as a SF2 RMIDI file which only refers to its own SoundFont bank,  
and the said bank contains **all and only** the necessary presets to play the file.
It is recommended to use DBNK of 0 for writing such files, but it is not required.

Writing self-contained RMIDI files is recommended, but not required.

### External File
An external file is defined as a SF2 RMIDI file which relies on a complete sound bank loaded as a fallback with the embedded sound bank only containing special sound effects, specific to the file.

The software not capable of loading two sound banks at once (the main one and the embedded one) may reject the file.

This type of file usually uses bank 1 or greater, but it may use bank 0.

## Recommendations for Writing RMIDI Files
The following recommendations are not required for file validity but are advised:
1. Trim the soundfont to include only presets and samples used in the file to save space.
2. Write a self-contained file to ensure that it will sound the same in every software.
3. Always include the DBNK chunk, even if the offset is 1.
4. Include the IENC chunk to ensure correct encoding is used.
5. Include the MENC chunk if the encoding is known, to help other software read the MIDI text events correctly.
6. Use the `utf-8` encoding for the metadata chunks if possible.
7. Use SoundFont3 compression if available to save space.

## Example Files
The directory [examples](examples) contains RMIDI Files for testing:

- `Field of Hopes and Dreams` - complete, level 4, self-contained file with IPIC chunk and metadata. Offset 0. Uses SF3 compression.
- `GRABBAG_EmbeddedSF2` - self-contained file with no `DBNK` chunk. Offset 1.
- `offset_5` - self-contained file with offset of 5.
- `Rock_test` - external file with no `DBNK` chunk. Offset 1, expects a full GM sound bank loaded at bank 0.
- `bachsb` - an RMIDI file without an embedded sound bank.
- `AWEBLOWN` - a DLS RMIDI file no `DBNK` chunk.
  Offset 1, expects a full GM sound bank loaded at bank 0. 
Software not capable of reading DLS should reject this file.


## Reference Implementation
Below is SpessaSynth implementation of the format in JavaScript, which may be useful for developers:

- [Loading the file](https://github.com/spessasus/SpessaSynth/blob/master/src/spessasynth_lib/midi_parser/midi_loader.js)
- [Writing the file](https://github.com/spessasus/SpessaSynth/blob/master/src/spessasynth_lib/midi_parser/rmidi_writer.js)
  - [Removing unused samples from the SoundFont](https://github.com/spessasus/SpessaSynth/blob/master/src/spessasynth_lib/soundfont/basic_soundfont/write_sf2/soundfont_trimmer.js)
- [Decoding and displaying the metadata](https://github.com/spessasus/SpessaSynth/blob/4243af6711261ba62ae78d8d1db532f2b766be75/src/website/js/music_mode_ui/music_mode_ui.js#L155)
