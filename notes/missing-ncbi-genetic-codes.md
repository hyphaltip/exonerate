# Missing NCBI genetic codes 24-33 (note, 2026-10-04)

Status: not implemented. Found while running MATPredict over BFD Ascomycota.

## Problem
`--geneticcode N` knows 17 built-in NCBI tables: 1-6, 9-16, 21-23
(`src/sequence/translate.c`, `Translate_get_builtin_genetic_code`, array
`code[17]`). NCBI now defines 10 more (24-33). For any of them exonerate stops:

```
** FATAL ERROR **: No built in genetic code corresponding to id [26]
exiting ...
```

Exit status 1. The man page also says "a number between 1 and 23 ... one of 17
built-in genetic code variants" (`doc/man/man1/exonerate.1`, line ~1029).

Impact seen: MATPredict v0.6.0 passed `--geneticcode 26` for 20 Alaninales yeast
genomes (*Pachysolen*, *Nakazawaea*; CUG = Ala). Every one failed. MATPredict
v0.6.1 now passes the table as the 64-letter string (see the workaround below).

## Checks made (exonerate 2.4.0 from the MATPredict pixi env)
- Every NCBI code 1-33 tried: 1-6, 9-16, 21-23 run; 24-31 and 33 exit 1
  (32 not tried).
- The 17 built-in strings match Biopython `Bio.Data.CodonTable` (NCBI) exactly,
  table by table.
- The longhand form already works: a 64-character string in NCBI TCAG order is
  accepted (`Translate_get_genetic_code`, `len == 64` branch). Test: a synthetic
  120-aa protein with 15 CTG codons, `protein2genome`:
  - `--geneticcode 1`: identity 87.50
  - `--geneticcode 12`: identity 87.50
  - `--geneticcode FFLLSSSSYY**CC*WLLLAPPPPHHQQRRRRIIIMTTTTNNKKSSRRVVVVAAAADDEEGGGG`
    (table 26): identity 100.00

So a caller can work around it today by passing the string. The built-in IDs are
still missing.

## Proposed change
1. Add tables 24-33 to `Translate_get_builtin_genetic_code`; change `code[17]`
   and the loop bound to 27 (better: `sizeof(code)/sizeof(code[0])`).
2. In `Translate_get_genetic_code`, the shorthand branch accepts only 1-2
   character IDs; 24-33 are 2 characters, so no change is needed there.
3. Update the man page: IDs 1-33, 27 built-in tables, list the new names.
4. Make the error list the valid IDs.
5. Add a test in `test/exonerate/` that runs `--geneticcode 26` on a small
   CTG-rich protein2genome case and checks the identity (and that 24-33 do not
   exit 1).

Tables to add (NCBI order TCAG; from Biopython `CodonTable.unambiguous_dna_by_id`,
stop codons as `*`). Check against the current NCBI gc.prt before committing:

| ID | Name | Code (TCAG order) |
|---|---|---|
| 24 | Pterobranchia Mitochondrial | `FFLLSSSSYY**CCWWLLLLPPPPHHQQRRRRIIIMTTTTNNKKSSSKVVVVAAAADDEEGGGG` |
| 25 | Candidate Division SR1 and Gracilibacteria | `FFLLSSSSYY**CCGWLLLLPPPPHHQQRRRRIIIMTTTTNNKKSSRRVVVVAAAADDEEGGGG` |
| 26 | Pachysolen tannophilus Nuclear | `FFLLSSSSYY**CC*WLLLAPPPPHHQQRRRRIIIMTTTTNNKKSSRRVVVVAAAADDEEGGGG` |
| 27 | Karyorelict Nuclear | `FFLLSSSSYYQQCC*WLLLLPPPPHHQQRRRRIIIMTTTTNNKKSSRRVVVVAAAADDEEGGGG` |
| 28 | Condylostoma Nuclear | `FFLLSSSSYY**CC*WLLLLPPPPHHQQRRRRIIIMTTTTNNKKSSRRVVVVAAAADDEEGGGG` |
| 29 | Mesodinium Nuclear | `FFLLSSSSYYYYCC*WLLLLPPPPHHQQRRRRIIIMTTTTNNKKSSRRVVVVAAAADDEEGGGG` |
| 30 | Peritrich Nuclear | `FFLLSSSSYYEECC*WLLLLPPPPHHQQRRRRIIIMTTTTNNKKSSRRVVVVAAAADDEEGGGG` |
| 31 | Blastocrithidia Nuclear | `FFLLSSSSYY**CCWWLLLLPPPPHHQQRRRRIIIMTTTTNNKKSSRRVVVVAAAADDEEGGGG` |
| 32 | Balanophoraceae Plastid | `FFLLSSSSYY*WCC*WLLLLPPPPHHQQRRRRIIIMTTTTNNKKSSRRVVVVAAAADDEEGGGG` |
| 33 | Cephalodiscidae Mitochondrial | `FFLLSSSSYYY*CCWWLLLLPPPPHHQQRRRRIIIMTTTTNNKKSSSKVVVVAAAADDEEGGGG` |

Caveat: in tables 27, 28 and 31 some codons are a stop in one context and an
amino acid in another. A 64-character string fixes one reading per codon (the
one shown above). This is the same limit every table-driven tool has.

## Funannotate (checked 2026-10-04)
- funannotate (`funannotate-live` 9a2c1d7, `aux_scripts/funannotate-p2g.py`
  line ~399): the exonerate p2g command has no `--geneticcode`, and the tblastn
  prefilter has no `-db_gencode`. `predict --table N` is checked
  (`genetic_codes.is_valid_table`) and used for translation elsewhere, but it is
  not passed to p2g.
- So funannotate never hits the fatal error. For a code-12 or code-26 genome,
  exonerate silently aligns with table 1 (CTG = Leu). Protein evidence at CTG
  codons scores as a mismatch (about 0.5-1% of residues in CUG-clade genes),
  which can drop alignments near the `--exonerate_pident` 80% cut-off.
- funannotate2 (`align.py` ~line 142): miniprot runs without `-T`, so the same
  silent table-1 translation.
- If funannotate passes the table through later, use the 64-letter string form
  for codes exonerate lacks (works with stock exonerate 2.4.0, as MATPredict
  v0.6.1 does), or the id once exonerate has tables 24-33. For miniprot, pass
  `-T N` (miniprot 0.18 has table 26).
