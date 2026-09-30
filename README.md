# squeeSat: Cross-Library Satellite Identification & Analysis

Protocol for searching satellite DNA homology across species using RepeatMasker, sequence extraction, and TAREAN/RepeatExplorer analysis.

---

## Prerequisites & Input Data

- **Reference:** Consensus sequences that satMiner / TAREAN / RepeatExplorer couldn't identify in a target species, but were present in at least one other species.
- **Library:** Raw reads from the target species where the satellite did not initially appear.

---

## Initial Processing (Steps 1–3)

> **Note:** Steps 1 to 3 only need to be run **once per library**. After initial processing, you can reuse the filtered output for steps 4–7 with multiple satellite candidates.

### 1. Run RepeatMasker

Run RepeatMasker across your dataset using `repeat_masker_run_big.py`:

```bash
repeat_masker_run_big.py list.txt reference 12
```

### 2. Join Output Files from a Single Library

Combine the chunked RepeatMasker `.out` files into a single master file:

```bash
# Generate file list
ls library.fasta*.out > lista_out.txt

# Join out files
rm_join_out.py

# Rename merged output
mv test.all.out library.fasta.out
```

> ⚠️ **TODO / Note:** The filename `lista_out.txt` is currently mandatory for `rm_join_out.py`. *Planned modification: allow custom input filenames.*

#### Example:
```bash
ls robuFM10.fasta*out > lista_out.txt
rm_join_out.py
mv test.all.out robuFM10.fasta.out
```

### 3. Filter Asterisks

Remove masked/overlapping regions marked with asterisks (`*`) in the output:

```bash
grep -v "*" library.out > library.out.noasterisk
```

> ❓ **Question:** Is this asterisk filtering step strictly necessary? *(To be verified)*

#### Example:
```bash
grep -v "*" robuFM10.fasta.out > robuFM10.fasta.out.noasterisk
```

---

## Extract & Re-cluster Satellites (Steps 4–7)

> 💡 **Automated Alternative:** Steps 4 through 6 can be automated using `sat_cross_libraries.py`:
> ```bash
> sat_cross_libraries.py robuFM10.fasta robuFM10.fasta.out.noasterisk lista_sats.txt
> ```

---

### Manual Pipeline (Steps 4–6)

#### 4. Extract Reads with Homology to a Specific Satellite

Filter the joined output for hits matching your target satellite name:

```bash
grep "SatelliteName" library.out.noasterisk > library.out.sat
```

##### Example:
```bash
grep "dsilveFR1CL58" robuFM10.fasta.out.noasterisk > robuFM10.dsilveFR1CL58.out
```

#### 5. Extract Unique Read IDs

Extract sequence identifiers and prepare paired-end read names (`/1` and `/2`):

```bash
awk '{print $5}' library.out.sat | sed 's/\//\t/g' | awk '{print $1}' | sort -u | awk '{print $1"/1\n"$1"/2"}' > selection.txt
```

##### Example:
```bash
awk '{print $5}' robuFM10.dsilveFR1CL58.out | sed 's/\//\t/g' | awk '{print $1}' | sort -u | awk '{print $1"/1\n"$1"/2"}' > robuFM10.dsilveFR1CL58.names
```

#### 6. Extract Reads and Sort by Header ID

Subset the original library FASTA file with the extracted read names and sort them by name:

```bash
# Subset FASTA
seqtk subseq library.fasta selection.txt > library_sel.fasta

# Sort FASTA by sequence ID (if required)
seqkit sort --by-name library_sel.fasta > library_sel.sort.fasta
```

##### Example:
```bash
seqtk subseq robuFM10.fasta robuFM10.dsilveFR1CL58.names > robuFM10.dsilveFR1CL58.fasta
seqkit sort --by-name robuFM10.dsilveFR1CL58.fasta > robuFM10.dsilveFR1CL58.sort.fasta
```

---

### 7. Run TAREAN on Galaxy Platform

1. Log into the **RepeatExplorer Galaxy** platform:
   👉 [https://repeatexplorer-elixir.cerit-sc.cz/galaxy](https://repeatexplorer-elixir.cerit-sc.cz/galaxy)
2. Open the **TAREAN** tool.
3. Configure the job settings:
   - **Paired-end Illumina reads:** Select `library_sel.sort.fasta` (e.g., `robuFM10.dsilveFR1CL58.sort.fasta`)
   - **Read sampling:** `No`
   - **Advanced options:** `Yes`
     - **Perform cluster merging:** `Yes`
     - **Use custom repeat database:** *Optional*
     - Leave remaining options as default.
4. Execute the tool.

