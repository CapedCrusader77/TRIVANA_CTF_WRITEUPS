# U - Digital Footprints 1

**Category:** OSINT  
**Points:** 150  
**Status:** Active

## Mission Briefing

A username can reveal more information than expected. This challenge is the first stage of a multi-part OSINT investigation.

The username provided by the challenge is:

```text
tri2026varna
```

The objective is to identify the correct online profile associated with this username and extract the information required for the next stage.

## Investigation

The supplied clue led to a Blogspot page named `flag-1.txt`. The page contained a flag-format clue together with an image.

The image was then investigated using reverse image search. This led to an official National Investigation Agency (NIA) record matching the person shown in the image.

The matching profile identifies:

- **Name:** Bexen Vincent
- **Aliases:** Isa, Easa
- **Organization:** ISIS
- **Case:** RC-03/2016/NIA/KOC

## Official Source

The information can be verified on the National Investigation Agency website:

[NIA — Bexen Vincent](https://nia.gov.in/node/3165)

The official record provides the identifying information needed to construct the challenge flag.

## Flag Construction

The required format is:

```text
TRIVARNA{Name_Organization_Date}
```

Using the information obtained during the investigation gives:

```text
Bexen_Vincent
ISIS
09_07_2016
```

## Final Flag

```text
TRIVARNA{Bexen_Vincent_ISIS_09_07_2016}
```
