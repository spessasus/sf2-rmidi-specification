# Official SF2 RMIDI Specification

## Preamble
MIDI files have long-faced a significant challenge: **different sounds on different devices.**

SF2 + MIDI combinations address this issue partially by ensuring that playing both files through an SF2-compliant synth results in the same sound being produced. 
The RMIDI format is not new; it was originally developed by Microsoft as a RIFF wrapper for MIDI files and later expanded by the MIDI Manufacturers Association to support embedding DLS soundfonts.
However, DLS is not widely used today, whereas the SoundFont2 (SF2) format serves a similar purpose and remains quite popular.

The SF2 RMIDI format integrates MIDI and SF2 files into a single file, augmented with additional metadata. 
This document serves as the official specification for this format.

This version of RMIDI was created by Zoltán Bacskó of [Falcosoft](https://falcosoft.hu) and implemented in [Falcosoft SoundFont Midi Player 6](https://falcosoft.hu/softwares.html#midiplayer). 
I am in contact with Zoltán, who [granted permission](https://www.vogons.org/viewtopic.php?p=1282710#p1282710) to use this as the official specification.

If you find any part of this specification unclear, please reach out via [this thread](https://www.vogons.org/viewtopic.php?p=1281224) or file a [GitHub issue](https://github.com/spessasus/sf2-rmidi-specification/issues/new) in this repository.

## Table of Contents
<!-- TOC -->
* [Official SF2 RMIDI Specification](#official-sf2-rmidi-specification)
  * [Preamble](#preamble)
  * [Table of Contents](#table-of-contents)
  * [Specification](#specification)
    * [Terminology](#terminology)
    * [Extension](#extension)
    * [RIFF Chunk](#riff-chunk)
    * [RMID File Structure](#rmid-file-structure)
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
    * [Program and Bank Correction](#program-and-bank-correction)
    * [Software Requirements](#software-requirements)
      * [Level 1](#level-1)
      * [Level 2](#level-2)
      * [Level 3](#level-3)
      * [Level 4](#level-4)
    * [Recommendations for Writing RMIDI Files](#recommendations-for-writing-rmidi-files)
    * [Example Files](#example-files)
    * [Reference Implementation](#reference-implementation)
<!-- TOC -->

## Specification
### Terminology
This specification assumes familiarity with the SoundFont2 format and the Standard MIDI File (SMF) format. 
Additional terminology used in this specification includes:

- **The software**: Refers to software compliant with this specification.
- **Bit**: The most basic data structure element, either 0 or 1.
- **Byte**: A data structure element of eight bits, with no defined meaning to those bits.
- **SoundFont**: A [SoundFont2](https://musescore.org/sites/musescore.org/files/2023-01/sfspec24.pdf) compliant binary.
- **Embedded SoundFont:** The SoundFont bank embedded in an RMIDI file.
- **Main SoundFont:** The regular SoundFont bank loaded by the software prior to loading the RMIDI file.
- **Bank**: MIDI controller 0 `Bank Select MSB` and the bank number of a soundfont preset.
- **RIFF**: Resource Interchange File Format. A file container format for storing data in tagged chunks.
- **Chunk**: The top-level division of a RIFF file.
- **Little Endian**: Byte ordering in memory with the least significant byte at the lowest address.
- **GM**: General MIDI system, ignoring all Bank select messages.
- **XG**: Yamaha eXtended General MIDI, an extension to the General MIDI standard created by Yamaha.
- **GS**: Roland General Standard, an extension to the General MIDI standard created by Roland.
- **Encoding**: Assigning numbers to graphical characters.
- **ASCII**: American Standard Code for Information Interchange, a character encoding standard for electronic communication.

### Extension
The file extension is `.rmi`, and the MIME type is `audio/rmid`. 
The file type should be referred to as `MIDI with embedded SF2` or `Embedded MIDI`.

### RIFF Chunk
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

### RMID File Structure
An RMIDI file consists of:
- `RIFF` chunk (main chunk)
  - `RMID` ASCII string
  - `data` chunk containing the complete MIDI file (MThd, MTrk, etc.)
  - Optional `LIST` chunk: Metadata for the file, similar to SF2's chunk
    - `INFO` ASCII string
    - [Inner chunks described here](#info-chunk)
  - `RIFF` chunk: Complete soundfont binary.
    The first four bytes of this chunk should be `sfbk`, indicating a soundfont. 
  [SoundFont3 format](https://github.com/FluidSynth/fluidsynth/wiki/SoundFont3Format) is allowed.

#### Handling Differences
When the file structure deviates from the above:
1. Any additional chunks should be ignored and preserved as-is.
2. If the chunk order differs from this specification, the file should be rejected. 
While software may support arbitrary chunk orders, compliance with this specification does not require it.
3. If no soundfont bank is present, the file should use the main soundfont and assume a bank offset of 0, ignoring the DBNK chunk.
4. If the soundfont bank uses the older DLS format, software not capable of reading DLS should reject the file. 
Software that supports DLS should use the contained DLS and assume a bank offset of 0, ignoring the DBNK chunk.

### INFO Chunk
The INFO chunk describes file metadata and the soundfont's bank offset and follows these rules:
- Any additional RIFF chunks within the INFO chunk should be ignored and preserved as-is.
- The chunk size must be even, as specified in the general RIFF structure.

The INFO chunk may contain the following optional chunks:
- `DBNK` chunk: Soundfont's bank offset. See [DBNK Chunk](#dbnk-chunk) for details.
- `IENC` chunk: Encoding used for the metadata chunks, string.
  Not case-sensitive, but lowercase is preferred (e.g., `utf-8`).
  [Software capable of reading the IENC chunk must support the following encodings](#ienc-chunk-requirements).
  Note that this field must use basic `ASCII` encoding.
- `MENC` chunk: Encoding hint for the MIDI file itself (the text events). The same string format as `IENC`.
- [Metadata chunks](#metadata-chunks)

### Metadata Chunks
Below are the defined chunks containing additional information about the song:
- `INAM` chunk: Song name.
- `ICOP` chunk: Copyright. String of any length.
- `IART` chunk: Artist (MIDI creator). String of any length.
- `ICRD` chunk: Creation date. String of any length.
- `IPRD` chunk: Album name. String of any length.
- `IPIC` chunk: Attached picture (e.g., album cover). Binary picture data. PNG or JPEG recommended.
- `IGNR` chunk: Song genre. String of any length.
- `ICMT` chunk: Comment/description. String of any length.
- `IENG` chunk: Engineer (soundfont creator). String of any length.
- `ISFT` chunk: Software used to create the file. String of any length.

#### Chunk Rules
The following rules apply to the INFO chunk:
1. The order of chunks in INFO is arbitrary.
2. Chunks of length 0 should be discarded.
3. Unknown INFO chunks should be ignored and preserved as-is.
4. If the `IENC` chunk is not specified, the software can use any encoding, but assuming `utf-8` is recommended.
5. If the `MENC` chunk is not specified, the software may try to automatically detect the MIDI encoding or let the user choose.
6. If the software can display the song's name, it should use either the track name in MIDI or the INAM chunk, preferring the INAM chunk.
7. Compatible software may ignore all INFO chunks **except the DBNK chunk** to remain compliant.

### IENC Chunk Requirements
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

### IPIC Chunk Requirements
For Level 4 compatibility, software must support the following image formats:
- Portable Network Graphics (PNG)
- Joint Photographic Experts Group (JPEG)

Other formats (e.g., `gif`, `webp`, `ico`) may also be supported but are not required.

### DBNK Chunk
The DBNK chunk is an optional RIFF chunk within the RMIDI INFO List.

It always has a length of two bytes,
with these bytes forming a 16-bit unsigned little-endian offset for the soundfont banks. 
If the chunk's length is not two bytes or the number is out of range, the file should be rejected.

Current boundaries are: **min: 0** and **max: 127**. 
The other byte is reserved for future use. 
If no DBNK is specified, an offset of **1** is assumed by default.

For general use, a bank offset of 0 is recommended as it allows bundling the soundfont and the MIDI without modification.

#### Bank Offset
The bank offset adjusts every bank in the soundfont, except for bank 128.

##### Bank offset 0
A bank offset of 0 has a few special characteristics:
1. If the software has a main soundfont, presets in the embedded soundfont override the main presets.
2. On drum channels, the bank is 0. For XG MIDIs, drum channels use bank 127.
3. The MIDI file can use the GM system and not contain any bank selects at all.
4. If the selected bank has not been found, the channel should fall back to the first preset with the given program number of the embedded soundfont, rather than the main one.
5. If the selected program has not been found, the channel should fall back to the first preset of the embedded soundfont, rather than the main one.

#### Other bank offsets
For example, for the bank offset of 1, the following rules apply:
1. Every bank in the soundfont is incremented by 1.
2. For drums, the bank is 1.
For XG MIDIs drum behavior is undefined; the software might expect bank 1 or bank 127. 
Using a bank offset of 0 in that case is recommended and defined.
3. The MIDI file must reflect the change as well: all bank select messages are incremented by 1 when compared to the original composition.
4. The MIDI must use valid banks and presets, 
as the software may fall back to the main soundfont
   instead of defaulting to the first preset of the embedded soundfont.
Missing presets in the MIDI sequence is undefined and should be avoided.
5. The system cannot be GM since the bank is ignored. GS or XG are valid. This requires sanitizing the MIDI and setting either GS or XG at the start.

For a DBNK value of 0, only constraint 3 applies.

The MIDI file must reflect this shift:
- For example, if DBNK is 1 and the MIDI requests preset 001:080, it should call a bank select message 2 instead of 1.
- If DBNK is 0, no offset is applied, and the MIDI remains unchanged.

### Program and Bank Correction
As stated in constraint 3, the MIDI must use valid banks and presets.
The system cannot be GM since the bank is ignored.
GS or XG are valid.
This requires sanitizing the MIDI and setting either GS or XG at the start.

### Software Requirements
Not all chunks in the file must be read for the file to play correctly. Software compatibility with the RMIDI format is categorized into levels:

#### Level 1
Basic RMIDI compatibility. The software must:
- Read and interpret the `RMID` ASCII string as the file indicator.
- Handle the `data` chunk containing the MIDI data.
- Process the `DBNK` chunk within the INFO chunk and correctly offset the soundfont (or a bank select messages in the MIDI) based on this value.
- Read the `RIFF` chunk with the soundfont data.

#### Level 2
This level requires basic interpretation of the `INFO` chunk. The software must:
- Read all Level 1 chunks.
- Interpret all metadata chunks (`INAM`, `IPRD`, `ICRD`, `ICOP`, etc.) as `ASCII` or `utf-8`.

#### Level 3
This level requires support for the `IENC` chunk. The software must:
- Read all Level 1 and Level 2 chunks.
- Interpret the `IENC` chunk and support the [required encodings](#ienc-chunk-requirements).

As of 2024-08-07, Falcosoft Midi Player meets this level of compatibility.

#### Level 4
This level requires support for the `IPIC` chunk. The software must:
- Read all Level 1, Level 2, and Level 3 chunks.
- Interpret the `IPIC` chunk and support the [required image formats](#ipic-chunk-requirements).

As of 2024-08-06, SpessaSynth meets this level of compatibility.

### Recommendations for Writing RMIDI Files
The following recommendations are not required for file validity but are advised:
1. Trim the soundfont to include only presets used in the file.
2. Ensure the MIDI file references used banks and programs to avoid undefined behavior from missing presets.
3. Always include the DBNK chunk, even if the offset is 1.
4. Include the IENC chunk to ensure correct encoding is used.
5. Include the MENC chunk if the encoding is known, to help other software read the MIDI text events correctly.
6. Use the `utf-8` encoding for the metadata chunks.
7. Omit metadata chunks rather than writing them with a length of 0 if not applicable.

### Example Files
The directory [examples](examples) contains RMIDI Files for testing.
Some contain offsets, some don't contain the DBNK chunk, and some contain everything, including the album cover.

### Reference Implementation
Below is SpessaSynth's implementation of the format in JavaScript, which may be useful for developers:

- [Loading the file](https://github.com/spessasus/SpessaSynth/blob/7e49fd5da62e433a3414bbc24bf012e9af96e84b/src/spessasynth_lib/midi_parser/midi_loader.js#L63)
- [Writing the file](https://github.com/spessasus/SpessaSynth/blob/master/src/spessasynth_lib/midi_parser/rmidi_writer.js)
- [Decoding and displaying the metadata](https://github.com/spessasus/SpessaSynth/blob/4243af6711261ba62ae78d8d1db532f2b766be75/src/website/js/music_mode_ui/music_mode_ui.js#L155)
