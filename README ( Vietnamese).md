# WGS Bacillus Assembly Workflow

## Giới thiệu

Repository này chứa quy trình phân tích dữ liệu WGS cho **Bacillus** từ dữ liệu Illumina paired-end, bao gồm:

- kiểm tra chất lượng dữ liệu thô
- trimming và lọc đọc
- kiểm tra lại sau tiền xử lý
- lắp ráp de novo
- đánh giá chất lượng assembly
- trích xuất trình tự 16S rRNA
- so sánh ANI với genome tham chiếu gần
- thử assembly thay thế bằng Unicycler
- reference-guided scaffolding bằng RagTag
- chuẩn bị môi trường để annotation bằng Bakta

Quy trình này phù hợp cho bài toán:
- lắp ráp genome isolate vi khuẩn
- kiểm tra nhanh chất lượng dữ liệu và assembly
- hỗ trợ định danh gần loài bằng 16S + ANI
- sắp xếp contig theo một reference gần để tạo scaffold dễ đọc hơn

## Tóm tắt quy trình

Luồng phân tích:

1. QC dữ liệu thô bằng FastQC + MultiQC  
2. Tiền xử lý reads bằng fastp  
3. QC lại dữ liệu sau trimming  
4. Lắp ráp de novo bằng SPAdes  
5. Đánh giá assembly bằng QUAST  
6. Tìm 16S rRNA bằng Barrnap và trích xuất FASTA bằng bedtools  
7. So sánh ANI với genome tham chiếu bằng FastANI  
8. Thử assembly khác bằng Unicycler  
9. Scaffold contigs theo reference bằng RagTag  
10. Đánh giá lại scaffold và kiểm tra số lượng ký tự `N`  
11. Chuẩn bị annotation bằng Bakta

## Cấu trúc thư mục đề xuất

```text
.
├── README.md
├── envs
│   ├── wgs_bacillus.yml
│   └── bakta.yml
├── data
│   ├── raw
│   │   └── <SAMPLE>_R1.fastq.gz
│   │   └── <SAMPLE>_R2.fastq.gz
│   └── ref
│       └── <REFERENCE>.fasta
├── results
│   ├── qc
│   │   ├── raw
│   │   ├── raw_multiqc
│   │   ├── trimmed
│   │   └── trimmed_multiqc
│   ├── trimmed
│   │   └── <SAMPLE>
│   ├── assembly
│   │   ├── spades
│   │   │   └── <SAMPLE>
│   │   ├── quast
│   │   │   └── <SAMPLE>
│   │   ├── unicycler
│   │   │   └── <SAMPLE>
│   │   └── ragtag
│   │       └── <SAMPLE>
│   ├── rrna
│   │   └── <SAMPLE>
│   ├── ani
│   │   └── <SAMPLE>
│   └── annotation
│       └── bakta
│           └── <SAMPLE>
└── scripts
    └── run_wgs_bacillus.sh
```

## Yêu cầu hệ thống

- Linux 64-bit
- Conda, Mamba hoặc Micromamba
- dữ liệu Illumina paired-end của một isolate
- một genome tham chiếu gần để chạy FastANI và RagTag
- dung lượng RAM đủ cho SPAdes/Unicycler
- cơ sở dữ liệu Bakta nếu muốn annotation

## Cài đặt môi trường

### Thiết lập Bioconda và Mamba

```bash
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --set channel_priority strict

conda install -n base -c conda-forge mamba -y
```

### Môi trường chính cho QC, assembly và so sánh genome

Tạo file `envs/wgs_bacillus.yml`:

```yaml
name: wgs_bacillus
channels:
  - conda-forge
  - bioconda
dependencies:
  - fastqc
  - multiqc
  - fastp
  - spades
  - quast
  - barrnap
  - bedtools
  - seqkit
  - fastani
  - unicycler
  - ragtag
```

Cài môi trường:

```bash
mamba env create -f envs/wgs_bacillus.yml
conda activate wgs_bacillus
```

### Môi trường annotation bằng Bakta

Tạo file `envs/bakta.yml`:

```yaml
name: bakta_env
channels:
  - conda-forge
  - bioconda
dependencies:
  - bakta
```

Cài môi trường:

```bash
mamba env create -f envs/bakta.yml
conda activate bakta_env
```

Tải cơ sở dữ liệu Bakta:

```bash
bakta_db download --output <BAKTA_DB_PARENT> --type full
```

Ví dụ:
- nếu tải vào `/data/db/bakta`, database thực dùng thường sẽ nằm trong một thư mục con kiểu `db/` hoặc `db-full/`
- sau đó truyền đường dẫn đó cho biến `BAKTA_DB`

## Khai báo biến chung

Bạn có thể chạy từng lệnh riêng lẻ, nhưng khuyến nghị dùng biến để README dễ tái sử dụng.

```bash
export PROJECT_DIR="/path/to/project"
export SAMPLE="<SAMPLE_ID>"
export THREADS=16
export TRIM_THREADS=8
export ASM_THREADS=8
export RAM_GB=16

export RAW_R1="${PROJECT_DIR}/data/raw/${SAMPLE}_R1.fastq.gz"
export RAW_R2="${PROJECT_DIR}/data/raw/${SAMPLE}_R2.fastq.gz"

export REF_FASTA="${PROJECT_DIR}/data/ref/<REFERENCE>.fasta"
export BAKTA_DB="/path/to/bakta_db"

mkdir -p \
  "${PROJECT_DIR}/results/qc/raw" \
  "${PROJECT_DIR}/results/qc/raw_multiqc" \
  "${PROJECT_DIR}/results/qc/trimmed" \
  "${PROJECT_DIR}/results/qc/trimmed_multiqc" \
  "${PROJECT_DIR}/results/trimmed/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/quast/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/unicycler/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}" \
  "${PROJECT_DIR}/results/rrna/${SAMPLE}" \
  "${PROJECT_DIR}/results/ani/${SAMPLE}" \
  "${PROJECT_DIR}/results/annotation/bakta/${SAMPLE}"
```

## Phân tích dữ liệu thô bằng FastQC và MultiQC

### Mục đích

- xem nhanh chất lượng reads thô
- kiểm tra adapter, base quality, GC, sequence duplication, overrepresented sequences
- tổng hợp nhiều báo cáo FastQC vào một báo cáo MultiQC

### Lệnh

```bash
fastqc \
  "${RAW_R1}" \
  "${RAW_R2}" \
  -o "${PROJECT_DIR}/results/qc/raw" \
  -t "${THREADS}"

multiqc \
  "${PROJECT_DIR}/results/qc/raw" \
  -o "${PROJECT_DIR}/results/qc/raw_multiqc"
```

## Tiền xử lý reads bằng fastp

### Mục đích

- tự phát hiện adapter cho paired-end
- cắt đuôi phải theo cửa sổ chất lượng
- lọc reads quá ngắn hoặc có quá nhiều base chất lượng thấp
- sửa mismatch trong vùng overlap của paired-end
- xuất cả báo cáo HTML và JSON

### Lệnh

```bash
fastp \
  -i "${RAW_R1}" \
  -I "${RAW_R2}" \
  -o "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R1.clean.fastq.gz" \
  -O "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R2.clean.fastq.gz" \
  --detect_adapter_for_pe \
  --cut_right \
  --cut_right_window_size 4 \
  --cut_right_mean_quality 20 \
  --qualified_quality_phred 20 \
  --unqualified_percent_limit 20 \
  --length_required 50 \
  --correction \
  --thread "${TRIM_THREADS}" \
  --html "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_fastp.html" \
  --json "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_fastp.json"
```

## Kiểm tra lại sau tiền xử lý

### Mục đích

- xác nhận trimming đã cải thiện chất lượng
- đối chiếu tổng quan giữa trước và sau xử lý
- thống kê nhanh số reads, tổng bases, chiều dài

### Lệnh

```bash
fastqc \
  "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R1.clean.fastq.gz" \
  "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R2.clean.fastq.gz" \
  -o "${PROJECT_DIR}/results/qc/trimmed" \
  -t "${THREADS}"

multiqc \
  "${PROJECT_DIR}/results/qc/trimmed" \
  -o "${PROJECT_DIR}/results/qc/trimmed_multiqc"

seqkit stats \
  "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R1.clean.fastq.gz" \
  "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R2.clean.fastq.gz"
```

## Lắp ráp de novo bằng SPAdes

### Mục đích

- lắp ráp genome isolate vi khuẩn từ paired-end reads sau tiền xử lý
- dùng riêng một output directory cho SPAdes để tránh trộn lẫn với file trung gian khác

### Lệnh

```bash
spades.py \
  --isolate \
  -1 "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R1.clean.fastq.gz" \
  -2 "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R2.clean.fastq.gz" \
  -o "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}" \
  -t "${ASM_THREADS}" \
  -m "${RAM_GB}"
```

Kiểm tra nhanh output:

```bash
ls -lh "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/"
```

> Ghi chú: workflow này tiếp tục dùng `contigs.fasta` để bám sát notebook gốc. Nếu muốn soát assembly đầu ra chuẩn theo SPAdes, bạn cũng nên xem thêm `scaffolds.fasta`.

## Đánh giá assembly bằng QUAST

### Mục đích

- xem số contig, tổng chiều dài, N50, L50
- nếu có reference, xem thêm misassemblies, genome fraction, số `N`

### Lệnh

```bash
quast.py \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  -o "${PROJECT_DIR}/results/assembly/quast/${SAMPLE}" \
  -t "${ASM_THREADS}"
```

## Tìm và trích xuất 16S rRNA

### Mục đích

- tìm vị trí rRNA trong assembly
- lọc riêng các feature 16S
- trích xuất 16S ra FASTA để phục vụ định danh hoặc BLAST ngoài pipeline

### Lệnh

```bash
barrnap \
  --kingdom bac \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  > "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_rrna.gff"
```

Lọc các dòng 16S:

```bash
grep "16S_rRNA" \
  "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_rrna.gff" \
  > "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.gff"
```

Xem nội dung:

```bash
cat "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.gff"
```

Trích xuất FASTA của 16S:

```bash
bedtools getfasta \
  -fi "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  -bed "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.gff" \
  -s \
  -name \
  > "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.fasta"
```

Thống kê 16S đã trích xuất:

```bash
seqkit stats "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.fasta"
```

## So sánh ANI với reference gần

### Mục đích

- so sánh toàn genome với reference gần để hỗ trợ định danh gần loài
- kiểm tra nhanh mức tương đồng với chủng tham chiếu

### Lệnh

```bash
fastANI \
  -q "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  -r "${REF_FASTA}" \
  -o "${PROJECT_DIR}/results/ani/${SAMPLE}/${SAMPLE}_vs_reference.ani.txt"
```

Xem kết quả:

```bash
cat "${PROJECT_DIR}/results/ani/${SAMPLE}/${SAMPLE}_vs_reference.ani.txt"
```

## Thử assembly thay thế bằng Unicycler

### Mục đích

- tạo một assembly thứ hai để đối chiếu với SPAdes
- hữu ích trong genome vi khuẩn vì Unicycler có thể tối ưu đồ thị assembly ngay cả khi chỉ dùng Illumina reads

### Lệnh

```bash
unicycler \
  -1 "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R1.clean.fastq.gz" \
  -2 "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R2.clean.fastq.gz" \
  -o "${PROJECT_DIR}/results/assembly/unicycler/${SAMPLE}" \
  -t "${ASM_THREADS}"
```

## Reference-guided scaffolding bằng RagTag

### Mục đích

- sắp xếp và định hướng contigs theo một genome tham chiếu gần
- tạo scaffold dễ đọc hơn và thuận lợi cho so sánh cấu trúc

### Lệnh

```bash
ragtag.py scaffold \
  "${REF_FASTA}" \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  -o "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}" \
  -t "${ASM_THREADS}"
```

Đánh giá scaffold bằng QUAST với reference:

```bash
quast.py \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/ragtag.scaffold.fasta" \
  -r "${REF_FASTA}" \
  -o "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/quast" \
  -t "${ASM_THREADS}"
```

Xem báo cáo text:

```bash
cat "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/quast/report.txt"
```

Thống kê độ dài và GC của scaffold:

```bash
seqkit fx2tab \
  -n -l -g \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/ragtag.scaffold.fasta"
```

Đếm tổng số ký tự `N` trong scaffold:

```bash
grep -o "N" \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/ragtag.scaffold.fasta" \
  | wc -l
```

## Annotation genome bằng Bakta

### Mục đích

- annotation nhanh và chuẩn hóa cho genome vi khuẩn hoặc plasmid
- nên chạy trên assembly cuối cùng bạn chọn dùng, ví dụ scaffold từ RagTag hoặc assembly tốt nhất theo QUAST

### Lệnh

```bash
conda activate bakta_env

bakta \
  --db "${BAKTA_DB}" \
  --threads "${ASM_THREADS}" \
  --output "${PROJECT_DIR}/results/annotation/bakta/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/ragtag.scaffold.fasta"
```

## Ý nghĩa nhanh của các tham số quan trọng

### Trong fastp

- `--detect_adapter_for_pe`: tự phát hiện adapter cho paired-end
- `--cut_right`: cắt từ trái sang phải bằng sliding window, gặp cửa sổ dưới ngưỡng thì cắt phần còn lại bên phải
- `--cut_right_window_size 4`: kích thước cửa sổ = 4 base
- `--cut_right_mean_quality 20`: ngưỡng trung bình trong cửa sổ là Q20
- `--qualified_quality_phred 20`: base đạt chuẩn nếu có Phred >= 20
- `--unqualified_percent_limit 20`: read bị loại nếu quá 20% base dưới chuẩn
- `--length_required 50`: loại các read ngắn dưới 50 bp sau xử lý
- `--correction`: sửa mismatch trong vùng overlap của pair

### Trong SPAdes

- `--isolate`: chế độ khuyến nghị cho isolate coverage cao
- `-t`: số luồng tính toán
- `-m`: RAM tối đa tính theo GB

### Trong Barrnap

- `--kingdom bac`: dùng profile phù hợp cho bacteria

### Trong RagTag

- `scaffold`: sắp xếp/orient contig theo reference gần

## Diễn giải đầu ra quan trọng

### FastQC / MultiQC

Dùng để xem:
- chất lượng theo vị trí base
- adapter contamination
- GC bias
- sequence duplication
- overrepresented sequences

### fastp report

Bao gồm:
- thống kê trước và sau lọc
- tỉ lệ Q20/Q30
- độ dài reads
- adapter bị loại
- số reads bị loại

### QUAST

Các chỉ số cần theo dõi:
- `# contigs`
- `Total length`
- `N50`
- `L50`
- `GC (%)`
- `# N's per 100 kbp`
- nếu có reference: `Genome fraction`, `# misassemblies`, `Duplication ratio`

### Barrnap + 16S FASTA

- xác định có 16S hay không
- hỗ trợ định danh bằng marker gene
- kiểm tra xem assembly có chứa đầy đủ rRNA mong đợi hay không

### FastANI

Output dạng tab, thường gồm:
- query genome
- reference genome
- ANI
- số fragment khớp hai chiều
- tổng số fragment query

### RagTag scaffold

- `ragtag.scaffold.fasta`: scaffold cuối
- `*.agp`: mô tả cách contig được ghép/orient
- QUAST sau RagTag giúp xem assembly có cải thiện về contiguity hay không, nhưng vẫn cần chú ý số `N` và nguy cơ bias theo reference

## Gợi ý thực hành tốt

- luôn giữ riêng thư mục `trimmed` và thư mục output của `spades`
- nên lưu cả HTML/JSON của fastp và HTML của FastQC/MultiQC
- nên so sánh ít nhất hai assembly đầu ra: SPAdes và Unicycler
- reference dùng cho RagTag nên là genome chất lượng cao và càng gần mẫu càng tốt
- không nên diễn giải scaffold theo reference như một “truth” tuyệt đối nếu reference quá xa
- nếu mục tiêu là định danh loài, ưu tiên đọc cả 16S và ANI thay vì chỉ dùng một chỉ dấu

## Ví dụ script chạy toàn bộ

Tạo file `scripts/run_wgs_bacillus.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

PROJECT_DIR="${1}"
SAMPLE="${2}"
REF_FASTA="${3}"
BAKTA_DB="${4:-}"

THREADS="${THREADS:-16}"
TRIM_THREADS="${TRIM_THREADS:-8}"
ASM_THREADS="${ASM_THREADS:-8}"
RAM_GB="${RAM_GB:-16}"

RAW_R1="${PROJECT_DIR}/data/raw/${SAMPLE}_R1.fastq.gz"
RAW_R2="${PROJECT_DIR}/data/raw/${SAMPLE}_R2.fastq.gz"

mkdir -p \
  "${PROJECT_DIR}/results/qc/raw" \
  "${PROJECT_DIR}/results/qc/raw_multiqc" \
  "${PROJECT_DIR}/results/qc/trimmed" \
  "${PROJECT_DIR}/results/qc/trimmed_multiqc" \
  "${PROJECT_DIR}/results/trimmed/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/quast/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/unicycler/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}" \
  "${PROJECT_DIR}/results/rrna/${SAMPLE}" \
  "${PROJECT_DIR}/results/ani/${SAMPLE}"

fastqc "${RAW_R1}" "${RAW_R2}" -o "${PROJECT_DIR}/results/qc/raw" -t "${THREADS}"
multiqc "${PROJECT_DIR}/results/qc/raw" -o "${PROJECT_DIR}/results/qc/raw_multiqc"

fastp \
  -i "${RAW_R1}" \
  -I "${RAW_R2}" \
  -o "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R1.clean.fastq.gz" \
  -O "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R2.clean.fastq.gz" \
  --detect_adapter_for_pe \
  --cut_right \
  --cut_right_window_size 4 \
  --cut_right_mean_quality 20 \
  --qualified_quality_phred 20 \
  --unqualified_percent_limit 20 \
  --length_required 50 \
  --correction \
  --thread "${TRIM_THREADS}" \
  --html "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_fastp.html" \
  --json "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_fastp.json"

fastqc \
  "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R1.clean.fastq.gz" \
  "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R2.clean.fastq.gz" \
  -o "${PROJECT_DIR}/results/qc/trimmed" \
  -t "${THREADS}"

multiqc "${PROJECT_DIR}/results/qc/trimmed" -o "${PROJECT_DIR}/results/qc/trimmed_multiqc"

seqkit stats \
  "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R1.clean.fastq.gz" \
  "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R2.clean.fastq.gz"

spades.py \
  --isolate \
  -1 "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R1.clean.fastq.gz" \
  -2 "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R2.clean.fastq.gz" \
  -o "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}" \
  -t "${ASM_THREADS}" \
  -m "${RAM_GB}"

quast.py \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  -o "${PROJECT_DIR}/results/assembly/quast/${SAMPLE}" \
  -t "${ASM_THREADS}"

barrnap \
  --kingdom bac \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  > "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_rrna.gff"

grep "16S_rRNA" \
  "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_rrna.gff" \
  > "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.gff"

bedtools getfasta \
  -fi "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  -bed "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.gff" \
  -s \
  -name \
  > "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.fasta"

seqkit stats "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.fasta"

fastANI \
  -q "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  -r "${REF_FASTA}" \
  -o "${PROJECT_DIR}/results/ani/${SAMPLE}/${SAMPLE}_vs_reference.ani.txt"

unicycler \
  -1 "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R1.clean.fastq.gz" \
  -2 "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R2.clean.fastq.gz" \
  -o "${PROJECT_DIR}/results/assembly/unicycler/${SAMPLE}" \
  -t "${ASM_THREADS}"

ragtag.py scaffold \
  "${REF_FASTA}" \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  -o "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}" \
  -t "${ASM_THREADS}"

quast.py \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/ragtag.scaffold.fasta" \
  -r "${REF_FASTA}" \
  -o "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/quast" \
  -t "${ASM_THREADS}"

seqkit fx2tab -n -l -g \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/ragtag.scaffold.fasta"

grep -o "N" \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/ragtag.scaffold.fasta" \
  | wc -l

if [[ -n "${BAKTA_DB}" ]]; then
  bakta \
    --db "${BAKTA_DB}" \
    --threads "${ASM_THREADS}" \
    --output "${PROJECT_DIR}/results/annotation/bakta/${SAMPLE}" \
    "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/ragtag.scaffold.fasta"
fi
```

Phân quyền và chạy:

```bash
chmod +x scripts/run_wgs_bacillus.sh

conda activate wgs_bacillus
bash scripts/run_wgs_bacillus.sh /path/to/project SAMPLE_ID /path/to/reference.fasta
```

Nếu muốn chạy thêm Bakta:

```bash
conda activate bakta_env
bash scripts/run_wgs_bacillus.sh /path/to/project SAMPLE_ID /path/to/reference.fasta /path/to/bakta_db
```

## Ghi chú cuối

- Nếu dữ liệu đầu vào là `.fq` không nén, chỉ cần đổi tên file trong biến `RAW_R1` và `RAW_R2`
- Nếu có nhiều sample, nên viết một vòng lặp bash hoặc workflow manager như Snakemake/Nextflow
- Nếu mục tiêu cuối là phân tích gene, nên quyết định rõ assembly nào là “final assembly” trước khi annotation
- Nếu reference không đủ gần, RagTag có thể tạo scaffold nhìn đẹp hơn nhưng không nhất thiết phản ánh đúng cấu trúc sinh học thật của isolate
