# Parallel Word Counting in a Text Corpus with MPI

## Team Information

- Repository: `mpi-word-counting-lab`
- Student(s): **[Add team member names here]**
- Course/Lab: **Parallel Word Counting in a Text Corpus with MPI**
- Date: May 2026

## Problem Description

The goal of this lab is to count how many times each word from `consulta.txt` appears in a text corpus formed by multiple files named `file_XXXX.txt`. After counting the occurrences, the program must report the top 10 most frequent query words in the corpus.

The lab compares three approaches:

1. A provided sequential baseline implementation.
2. A first MPI implementation, `mpi1.py`, using broadcast and static file distribution.
3. A second MPI implementation, `mpi2.py`, designed to reduce load imbalance by distributing files according to their size.

The dataset is generated using the provided `generator.py` script. The generated `dataset/` folder contains:

- `consulta.txt`, with one query word per line.
- Multiple `file_XXXX.txt` files, which form the text corpus.

## Repository Contents

```text
generator.py
baseline_secuencial.py
run_all.sh
mpi1.py
mpi2.py
Readme.md
.gitignore
```

The generated `dataset/` folder and the generated CSV files are not included in the repository because they can be reproduced by running the generator and the implementations.

## Environment and Execution Instructions

The experiments were executed using Docker with the image:

```bash
augustosalazar/slim-mpi:2
```

The dataset was generated with:

```bash
docker run --rm -v "${PWD}:/app" augustosalazar/slim-mpi:2 python /app/generator.py
```

The sequential baseline was executed with:

```bash
docker run --rm -v "${PWD}:/app" augustosalazar/slim-mpi:2 python /app/baseline_secuencial.py
```

The number of cores available inside the Docker container was checked with:

```bash
docker run --rm augustosalazar/slim-mpi:2 nproc
```

The container reported:

```text
7 available cores
```

Since the lab required testing with 8 MPI processes, the 8-process executions were run with `--oversubscribe`.

Example command for `mpi1.py`:

```bash
docker run --rm -v "${PWD}:/app" augustosalazar/slim-mpi:2 mpiexec --allow-run-as-root -n 4 python /app/mpi1.py
```

Example command for 8 processes:

```bash
docker run --rm -v "${PWD}:/app" augustosalazar/slim-mpi:2 mpiexec --allow-run-as-root --oversubscribe -n 8 python /app/mpi1.py
```

The same structure was used for `mpi2.py`.

## Experimental Plan

The experiment was divided into four stages:

1. Run the sequential baseline and record its execution time and output.
2. Implement and test `mpi1.py` using static file distribution.
3. Run `mpi1.py` with `p = 1, 2, 4, 8`, performing at least three runs per configuration.
4. Implement and test `mpi2.py` using a load-balancing strategy based on file size.
5. Repeat the same experimental procedure for `mpi2.py`.
6. Compare both MPI implementations using total execution time, speedup, efficiency, and load balance.

The metrics were computed as:

```text
Speedup = Tseq / Tp
Efficiency = Speedup / p
```

Where:

- `Tseq` is the sequential execution time.
- `Tp` is the average execution time for `p` MPI processes.
- `p` is the number of MPI processes.

## Sequential Baseline

The provided sequential implementation, `baseline_secuencial.py`, was used as the correctness and timing reference. It reads the query words from `consulta.txt`, scans all corpus files sequentially, counts the occurrences of the query words, and prints the top 10 most frequent words.

### Sequential Baseline Timing

```text
Tseq = 30.263341 seconds
```

### Sequential Output

The sequential baseline processed:

```text
Files processed: 3000
Total tokens read: 44,951,458
Total occurrences found: 3,631,778
```

Top 10 words:

| Rank | Word | Count |
|---:|---|---:|
| 1 | a | 785774 |
| 2 | para | 392156 |
| 3 | sus | 228913 |
| 4 | otros | 105530 |
| 5 | ante | 99832 |
| 6 | unos | 88794 |
| 7 | otra | 83901 |
| 8 | vosotros | 61617 |
| 9 | mios | 58420 |
| 10 | tuya | 56635 |

## MPI Version 1: Static File Distribution

The first MPI implementation, `mpi1.py`, follows this strategy:

1. Rank 0 reads `consulta.txt`.
2. Rank 0 broadcasts the query words to all processes using `comm.bcast`.
3. Rank 0 obtains the list of `file_*.txt` files.
4. Files are distributed statically among processes using slicing: `all_files[i::size]`.
5. Each process counts the occurrences of the query words in its assigned files.
6. Partial counters are gathered in rank 0.
7. Rank 0 builds the global result and prints the top 10 words.
8. Each process reports the number of assigned files and its local processing time.

This version distributes the same number of files per process, but it does not consider file size or number of tokens per file.

### MPI Version 1 Timing Results

| Processes | Run 1 (s) | Run 2 (s) | Run 3 (s) | Average Time (s) | Speedup | Efficiency |
|---:|---:|---:|---:|---:|---:|---:|
| 1 | 28.583822 | 28.116389 | 27.816959 | 28.172390 | 1.07 | 1.07 |
| 2 | 14.526825 | 14.191730 | 14.621213 | 14.446589 | 2.09 | 1.05 |
| 4 | 9.522224 | 10.725020 | 10.626242 | 10.291162 | 2.94 | 0.74 |
| 8 | 8.481068 | 8.422153 | 8.560188 | 8.487803 | 3.57 | 0.45 |

## Load Imbalance Evidence in MPI Version 1

The first MPI version assigns the same number of files to each process, but this does not guarantee the same workload. Some files contain more tokens than others, so processes may receive the same number of files while processing different amounts of text.

For example, with 8 MPI processes, each rank received 375 files, but the number of tokens was different:

| Rank | Assigned Files | Local Tokens | Local Time (s) |
|---:|---:|---:|---:|
| 0 | 375 | 5,018,405 | 7.319489 |
| 1 | 375 | 5,150,686 | 7.539032 |
| 2 | 375 | 6,161,885 | 8.401884 |
| 3 | 375 | 5,488,544 | 7.826678 |
| 4 | 375 | 5,769,736 | 8.080510 |
| 5 | 375 | 5,914,137 | 8.207266 |
| 6 | 375 | 5,412,053 | 7.836693 |
| 7 | 375 | 6,036,012 | 8.249075 |

This shows that static distribution by number of files caused load imbalance. Rank 2 processed significantly more tokens than rank 0, even though both received the same number of files.

## MPI Version 2: Load-Balancing Improvement

The second MPI implementation, `mpi2.py`, was designed to reduce the load imbalance observed in `mpi1.py`.

Its strategy is:

1. Rank 0 reads `consulta.txt`.
2. Rank 0 broadcasts the query words to all processes.
3. Rank 0 obtains the list of `file_*.txt` files.
4. Rank 0 calculates the size of each file.
5. Files are sorted from largest to smallest.
6. A greedy distribution assigns each file to the process with the smallest accumulated estimated load.
7. Each process counts locally.
8. Partial results are gathered in rank 0.
9. Rank 0 prints the global top 10 and process summaries.

This version attempts to balance the work using file size as an approximation of processing cost.

### MPI Version 2 Timing Results

| Processes | Run 1 (s) | Run 2 (s) | Run 3 (s) | Average Time (s) | Speedup | Efficiency |
|---:|---:|---:|---:|---:|---:|---:|
| 1 | 34.011210 | 32.186914 | 31.316714 | 32.504946 | 0.93 | 0.93 |
| 2 | 20.668290 | 19.151954 | 18.878210 | 19.566151 | 1.55 | 0.77 |
| 4 | 13.557397 | 14.197596 | 15.780294 | 14.511762 | 2.09 | 0.52 |
| 8 | 11.481889 | 14.184700 | 12.191272 | 12.619287 | 2.40 | 0.30 |

### Load Balance Evidence in MPI Version 2

With 8 MPI processes, `mpi2.py` produced a much more balanced distribution by estimated file size:

| Rank | Assigned Files | Estimated Size | Local Tokens | Local Time (s) |
|---:|---:|---:|---:|---:|
| 0 | 376 | 34,381,755 | 5,620,840 | 7.400400 |
| 1 | 375 | 34,381,731 | 5,619,966 | 7.321078 |
| 2 | 374 | 34,381,770 | 5,617,956 | 7.639206 |
| 3 | 375 | 34,381,802 | 5,618,676 | 7.585041 |
| 4 | 375 | 34,381,787 | 5,618,493 | 7.376246 |
| 5 | 375 | 34,381,696 | 5,618,576 | 7.557116 |
| 6 | 375 | 34,381,794 | 5,618,127 | 7.511503 |
| 7 | 375 | 34,381,724 | 5,618,824 | 7.561180 |

Compared with `mpi1.py`, the number of tokens per rank became more similar. This indicates that the file-size-based distribution reduced load imbalance.

## Comparison Between MPI Version 1 and MPI Version 2

| Processes | MPI 1 Average (s) | MPI 1 Speedup | MPI 1 Efficiency | MPI 2 Average (s) | MPI 2 Speedup | MPI 2 Efficiency |
|---:|---:|---:|---:|---:|---:|---:|
| 1 | 28.172390 | 1.07 | 1.07 | 32.504946 | 0.93 | 0.93 |
| 2 | 14.446589 | 2.09 | 1.05 | 19.566151 | 1.55 | 0.77 |
| 4 | 10.291162 | 2.94 | 0.74 | 14.511762 | 2.09 | 0.52 |
| 8 | 8.487803 | 3.57 | 0.45 | 12.619287 | 2.40 | 0.30 |

## Correctness Check

Both MPI versions produced the same top 10 results as the sequential baseline when the result CSV files were inspected. The top 10 words were:

| Rank | Word | Count |
|---:|---|---:|
| 1 | a | 785774 |
| 2 | para | 392156 |
| 3 | sus | 228913 |
| 4 | otros | 105530 |
| 5 | ante | 99832 |
| 6 | unos | 88794 |
| 7 | otra | 83901 |
| 8 | vosotros | 61617 |
| 9 | mios | 58420 |
| 10 | tuya | 56635 |

The total number of occurrences found was also consistent:

```text
Total occurrences found: 3,631,778
```

## Analysis

### Did the first MPI implementation improve execution time compared to the sequential baseline?

Yes. The sequential baseline execution time was `30.263341` seconds. The first MPI implementation reduced the average execution time to `14.446589` seconds with 2 processes, `10.291162` seconds with 4 processes, and `8.487803` seconds with 8 processes. Therefore, `mpi1.py` achieved a clear performance improvement over the sequential baseline.

### Was the observed speedup linear?

No. The speedup improved as the number of processes increased, but it was not linear. With 8 processes, the speedup was `3.57`, not close to the ideal speedup of 8. This indicates that the program is affected by overheads such as file I/O, communication, synchronization, and process scheduling.

### Is there evidence of load imbalance? How was it observed?

Yes. In `mpi1.py`, the processes received the same number of files, but they did not process the same number of tokens. For example, with 8 processes, rank 0 processed 5,018,405 tokens while rank 2 processed 6,161,885 tokens. This difference means that some processes had more work than others, even though the number of assigned files was equal.

### Did the second implementation reduce load imbalance?

Yes. The second implementation reduced the load imbalance by distributing files according to their estimated size. With 8 processes, the estimated sizes were almost equal across ranks, and the number of tokens processed by each rank was also much more similar than in `mpi1.py`.

### Did the improved distribution strategy produce a real performance improvement?

Not in total execution time. Although `mpi2.py` improved load balance, it did not outperform `mpi1.py`. With 8 processes, `mpi1.py` had an average execution time of `8.487803` seconds, while `mpi2.py` had an average execution time of `12.619287` seconds. The additional cost of calculating file sizes, sorting files, and building a balanced distribution likely increased the total runtime.

### What limitations affected the experiment?

The main limitation was the execution environment. The Docker container reported only 7 available cores, but the lab required testing with 8 MPI processes. Therefore, the 8-process runs were executed using `--oversubscribe`, which may affect performance because the number of MPI processes exceeds the number of available cores.

Another limitation is that file size is only an approximation of workload. Although it is useful for balancing the distribution, actual processing time can still vary due to disk I/O, OS scheduling, Docker overhead, and differences in tokenization cost.

## Conclusions

The parallel implementations produced correct results and matched the sequential baseline output. The first MPI version improved execution time significantly compared to the sequential baseline, especially as the number of processes increased. The best performance was obtained with `mpi1.py` using 8 processes, reaching an average execution time of `8.487803` seconds and a speedup of `3.57`.

The main problem observed in `mpi1.py` was load imbalance. Even though each process received the same number of files, the number of tokens processed by each rank was different. This showed that static distribution by file count was not enough to guarantee equal workload.

The second MPI version improved load balance by distributing files according to their estimated size. This produced more similar workloads among processes. However, the improvement in load balance did not result in better total execution time. The extra overhead of computing file sizes, sorting files, and assigning files greedily made `mpi2.py` slower than `mpi1.py` in the measured experiments.

Based on the evidence, the best-performing version was `mpi1.py`, while `mpi2.py` demonstrated better load balance but lower overall performance. Therefore, the improved distribution strategy was useful for analyzing load balance, but it did not produce a real performance improvement in this experiment.