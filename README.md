# Official SF2 RMIDI Specification
Original format idea by Zoltán Bacskó of [Falcosoft](https://falcosoft.hu), further expanded by spessasus.
Specification written by spessasus with the help of Zoltán.

Revision 1.17
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
* [SF2 RMIDI File Specification](#sf2-rmidi-file-specification)
  * [File Structure](#file-structure)
    * [Handling Differences](#handling-differences)
  * [INFO Chunk](#info-chunk)
    * [Metadata Chunks](#metadata-chunks)
    * [Chunk Rules](#chunk-rules)
      * [IENC Chunk Requirements](#ienc-chunk-requirements)
      * [IPIC Chunk Requirements](#ipic-chunk-requirements)
      * [DBNK Chunk](#dbnk-chunk)
  * [Bank Offset](#bank-offset)
    * [Bank offset 0](#bank-offset-0)
    * [Other bank offsets](#other-bank-offsets)
  * [Loop points](#loop-points)
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
- **Embedded SoundFont:** The SoundFont bank embedded in an RMIDI file.
- **Main SoundFont:** The regular SoundFont bank loaded by the software prior to loading the RMIDI file.
- **Bank**: MIDI controller 0 `Bank Select MSB` and the bank number of a soundfont preset.
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

### Handling Differences
When the file structure deviates from the above:
1. Any additional chunks should be ignored and preserved as-is.
2. If the chunk order differs from this specification, the file should be rejected.
3. If no soundfont bank is present, the file should use the main soundfont and assume a bank offset of 0, ignoring the DBNK chunk.
4. If the soundfont bank uses the older DLS format, software not capable of reading DLS should reject the file. 
Software that supports DLS should use the contained DLS and assume a bank offset of **1**, ignoring the DBNK chunk.

The last two rules ensure backwards compatibility with the older RMIDI format.

## INFO Chunk
The INFO chunk describes file metadata and the soundfont's bank offset and follows these rules:
- Any additional RIFF chunks within the INFO chunk should be ignored and preserved as-is.
- The chunk size must be even, as specified in the general RIFF structure.
- The INFO chunk is optional.

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
- `IPRD` or `IALB` chunk: Album name. String of any length. Can be used interchangeably. If both exist in the file, the software should use `IALB`.
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

It always has a length of two bytes,
with these bytes forming a **16-bit, unsigned, little-endian** offset for the soundfont banks. 
If the chunk's length is not two bytes or the number is out of range, the file should be rejected.

Current boundaries are: **minimum: 0** and **maximum: 127**. 
The other byte is reserved for future use. 
If no DBNK is specified, an offset of **1** is assumed by default.
If the file is not an SF2 RMIDI file (doesn't contain a sound bank or contains DLS), 
an offset of 0 is assumed and DBNK is to be ignored.

For general use, a bank offset of 0 is recommended as it allows bundling the soundfont and the MIDI without modification.

## Bank Offset
The bank offset adjusts every bank in the soundfont, except for bank 128.

### Bank offset 0
A bank offset of 0 has a few special characteristics:
1. If the software has a main soundfont, presets in the embedded soundfont override the main presets.
2. On drum channels, the bank is to be ignored for GM/GS MIDIs. For XG MIDIs, drum channels, bank 127 indicates a drum channel.
3. The MIDI file can use the GM system and not contain any bank selects at all.
4. If the selected bank has not been found, the channel should fall back to the first preset with the given program number of the embedded soundfont, rather than the main one.
5. If the selected program has not been found, the channel should fall back to the main soundfont. If the software does not have a Main soundfont loaded, the channel should fall back to the first preset within the embedded soundfont.
6. If the program has not been found within the main soundfont, the program should fall back to the first preset of the main soundfont.

### Other bank offsets
For example, for the bank offset of 1, the following rules apply:
1. Every bank in the embedded SoundFont is incremented by 1.
2. For drums, the bank is 1.
For XG MIDIs drum behavior is undefined; the software might expect bank 1 or bank 127. 
Using a bank offset of 0 in that case is recommended and defined.
3. The MIDI file must reflect the change as well: all bank select messages are incremented by 1 when compared to the original composition.
4. If the MIDI file references a bank and program combination does not exist within the embedded SoundFont bank, 
the software should fall back to the main sound bank and use it as normal.
5. The system cannot be GM since the bank is ignored. GS or XG are valid. This requires sanitizing the MIDI and setting either GS or XG at the start.


The MIDI file must reflect this bank shift:
- For example, if DBNK is 1 and the original MIDI requests preset 001:080, it should call a bank select message 2 instead of 1.
- If DBNK is 0, no offset is applied, and the MIDI remains unchanged.

## Loop points
As there are many implementations of loop points within various MIDI files and none of them are standardized,
this section of the document describes the recommended loop point behavior for the SF2 RMIDI format:

1. Loop points are optional. 
   If there are none, the file should be played normally.
   If the software's loop mode is enabled, 
   it should default to the first Note On as start point,
   and last note Off message as the end point.
2. Loop points are defined using Controller Change messages within the embedded MIDI data. 
   These loop points apply to the entire file, regardless of the MIDI channel the messages are in.
3. If loop points are present in the file, 
   there must always be an even number of them (two points make one loop).
   If that's not the case, all loop points must be rejected.
4. Start loop point is defined using CC#116.
   The controller value specifies the number of loops for this loop point pair.
   A value of zero is equivalent to infinite.
5. End loop point is defined using CC#117. The controller value is not used, but setting it to value of 127 is recommended.
6. Overlapping (nesting) loops is ILLEGAL. If any nested loops are encountered, all loops within the file must be rejected.
7. If a start loop point does not have a matching loop end point, all loop points in the file must be rejected.
8. If any other non-standard type of loop point is detected within the file while the standardized loop points are present, 
   the software must use the standardized loop points.

All SF2 RMIDI compatible players with loop point capability should support these.
If the software writing an RMIDI file has detected loop points in the original file,
it should translate them to this standardized format.

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

## Types of RMIDI Files
There are currently two distinct types of RMIDI files which vary in their use cases.

### Self-Contained File
A self-contained file is defined as a SF2 RMIDI file which only refers to its own SoundFont bank,  
and the said bank contains **all and only** the necessary presets to play the file.
It is recommended to use DBNK of 0 for writing such files, but it is not required.

Writing self-contained RMIDI files is recommended, but not required.

As such,
the MIDI file must be checked by the software
writing the file for any program changes that do not have a corresponding preset within the sound bank and correct them.
The software also must remove all unnecessary samples,
instruments and presets from the sound bank to keep the file size to a minimum.

### External File
An external file is defined as a SF2 RMIDI file which relies on a complete sound bank loaded as a fallback with the embedded sound bank only containing special sound effects, specific to the file.

The software not capable of loading two sound banks at once (the main one and the embedded one) may reject the file.

This type of file usually uses bank 1 or greater, but it may use bank 0.

## Recommendations for Writing RMIDI Files
The following recommendations are not required for file validity but are advised:
1. Trim the soundfont to include only presets used in the file to save space.
2. Write a self-contained file to ensure that it will sound the same in every software.
3. Always include the DBNK chunk, even if the offset is 1.
4. Include the IENC chunk to ensure correct encoding is used.
5. Include the MENC chunk if the encoding is known, to help other software read the MIDI text events correctly.
6. Use the `utf-8` encoding for the metadata chunks.
7. Use SoundFont3 compression if available to save space.

## Example Files
The directory [examples](examples) contains RMIDI Files for testing.

- `Field of Hopes and Dreams` - complete, level 4, self-contained file with IPIC chunk and metadata. Offset 0. Uses SF3 compression.
- `GRABBAG_EmbeddedSF2` - self-contained file with no `DBNK` chunk. Offset 1.
- `offset_5` - self-contained file with offset of 5.
- `Rock_test` - external file with no `DBNK` chunk. Offset 1, expects a full GM sound bank loaded at bank 0.

## Reference Implementation
Below is SpessaSynth implementation of the format in JavaScript, which may be useful for developers:

- [Loading the file](https://github.com/spessasus/SpessaSynth/blob/7e49fd5da62e433a3414bbc24bf012e9af96e84b/src/spessasynth_lib/midi_parser/midi_loader.js#L63)
- [Writing the file](https://github.com/spessasus/SpessaSynth/blob/master/src/spessasynth_lib/midi_parser/rmidi_writer.js)
- [Decoding and displaying the metadata](https://github.com/spessasus/SpessaSynth/blob/4243af6711261ba62ae78d8d1db532f2b766be75/src/website/js/music_mode_ui/music_mode_ui.js#L155)
