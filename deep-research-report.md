# Quy trình lắp ráp và chú giải bộ gen *Bacillus* (WGS)

**Tóm tắt:** Đây là quy trình phân tích dữ liệu Whole Genome Sequencing (WGS) của mẫu vi khuẩn. Các bước chính bao gồm: kiểm tra chất lượng dữ liệu đầu vào, làm sạch adapter và lọc reads, thực hiện lắp ráp de novo, đánh giá chất lượng lắp ráp, phát hiện gen rRNA (bao gồm 16S) và trích xuất trình tự tương ứng, so sánh độ tương đồng genome (ANI) với genome tham chiếu để định danh loài, và cuối cùng là chú giải genome bằng công cụ Bakta. Hình dưới đây minh họa tổng quát luồng công việc; bảng tóm tắt các bước được trình bày bên dưới; và các phần chi tiết mô tả từng công cụ, lệnh, đầu vào/đầu ra, cùng các lưu ý thực thi được giải thích bên dưới.

```mermaid
flowchart LR

    subgraph QC_RAW["QC dữ liệu thô"]
        A["FastQC (raw)"] --> B["MultiQC (raw)"]
    end

    subgraph PROCESSING["Xử lý reads"]
        B --> C["fastp (lọc reads)"]
    end

    subgraph QC_CLEAN["QC dữ liệu sạch"]
        C --> D["FastQC (sau lọc)"]
        D --> E["MultiQC (sau lọc)"]
        E --> F["SeqKit stats (reads sạch)"]
    end

    F --> G["SPAdes (assembly)"]
    G --> H["QUAST (đánh giá assembly)"]
    H --> I{"Assembly đạt yêu cầu?"}

    I -->|Đạt| J["Barrnap (dự đoán rRNA)"]
    I -->|Không đạt| K["Unicycler (phương án thay thế)"]

    K --> L["QUAST (đánh giá Unicycler)"]
    L --> M{"Unicycler tốt hơn?"}

    M -->|Có| J
    M -->|Không| N{"Có genome tham chiếu?"}

    N -->|Có| O["RagTag (scaffolding)"]
    N -->|Không| J

    O --> P["QUAST (đánh giá scaffold)"]
    P --> J

    subgraph RRNA["Phân tích rRNA"]
        J --> Q["Trích xuất 16S bằng grep và bedtools"]
        Q --> R["SeqKit stats (16S)"]
    end

    J --> S["FastANI (so sánh genome)"]
    R --> T["Bakta (chú giải genome)"]
    S --> T

    style K stroke-dasharray: 5 5
    style L stroke-dasharray: 5 5
    style M stroke-dasharray: 5 5
    style N stroke-dasharray: 5 5
    style O stroke-dasharray: 5 5
    style P stroke-dasharray: 5 5
```

**Giải thích sơ đồ:** Các bước màu liền mạch (QC, xử lý, assembly, chú giải) là bắt buộc. Các bước tác vụ phụ trợ hoặc thay thế (Unicycler, RagTag) được đánh dấu đường đứt nét và chỉ thực hiện khi cần (xem mục *Quyết định điều kiện* bên dưới).  

## Chuẩn bị môi trường phần mềm

Quy trình yêu cầu cài đặt nhiều công cụ bioinformatics phổ biến. Cách đơn giản là tạo một môi trường `conda` mới, ví dụ:

```bash
conda create -n wgs_bacillus -c conda-forge -c bioconda \
    fastqc multiqc fastp seqkit spades quast barrnap bedtools \
    fastani unicycler ragtag bakta
conda activate wgs_bacillus
```

*(Các phiên bản cụ thể có thể được chỉ định, tùy ý. Nếu Bakta yêu cầu ban đầu, có thể cần chạy `bakta_db install` để tải database; Barrnap đã có sẵn các mô hình HMM rRNA sau khi cài)*.  

Hoặc sử dụng tệp YAML ví dụ (ở dự án có thể kèm file `environment.yml`) như sau:

```yaml
name: wgs_bacillus
channels:
  - conda-forge
  - bioconda
dependencies:
  - python=3.9
  - fastqc
  - multiqc
  - fastp
  - seqkit
  - spades
  - quast
  - barrnap
  - bedtools
  - fastani
  - unicycler
  - ragtag
  - bakta
```

Đảm bảo đã cài `samtools` nếu cần (một số công cụ có thể phụ thuộc), và các database như *Barrnap* (sẵn trong conda) hay *Bakta DB* (xem hướng dẫn của Bakta). Ngoài ra, cần chuẩn bị tập tin FASTA của genome tham chiếu (ví dụ *Bacillus velezensis* FZB42) nếu sử dụng FastANI hoặc RagTag.

## Cấu trúc thư mục

Một ví dụ về cấu trúc thư mục cho quy trình này:

```
project/
├── RawRead/              # Thư mục chứa reads thô (R1 và R2 FASTQ)
│   ├── sample_R1.fastq.gz
│   └── sample_R2.fastq.gz
├── QC/                   # Thư mục kết quả kiểm tra chất lượng
│   ├── FastQC_raw/       # Báo cáo FastQC của dữ liệu thô
│   ├── MultiQC_raw/      # Báo cáo MultiQC (tổng hợp FastQC) cho dữ liệu thô
│   ├── FastQC_clean/     # Báo cáo FastQC của dữ liệu sau lọc
│   └── MultiQC_clean/    # Báo cáo MultiQC (tổng hợp) cho dữ liệu đã lọc
├── sample/               # Kết quả xử lý cho mẫu (ví dụ thư mục "L1")
│   ├── sample_R1.clean.fastq    # Reads sạch R1
│   ├── sample_R2.clean.fastq    # Reads sạch R2
│   ├── sample_fastp.html        # Báo cáo HTML của fastp
│   ├── sample_fastp.json        # Báo cáo JSON của fastp
│   ├── contigs.fasta            # File contigs kết quả SPAdes
│   ├── L1_unicycler/            # Thư mục đầu ra của Unicycler
│   ├── L1_ragtag/               # Thư mục đầu ra của RagTag
│   └── ...                      # Các file tạm và logs khác
├── README.md             # README giải thích quy trình này
└── ...                   # Các file dữ liệu, báo cáo khác
```

Thay thế `project/`, `RawRead/`, `sample/` bằng tên thực của dự án và mẫu tương ứng. Mỗi bước chạy ra kết quả ở thư mục thích hợp như trên.

## Bảng tóm tắt các bước

| Bước | Công cụ        | Mục đích                                                     | Đầu vào chính                  | Đầu ra chính                    | Thực hiện khi               | Tham số chính           |
|:----:|:--------------|:------------------------------------------------------------|:------------------------------|:-------------------------------|:---------------------------|:-------------------------|
| 1    | FastQC        | Kiểm tra chất lượng ban đầu của reads                         | Tất cả file FASTQ thô         | HTML/ZIP báo cáo FastQC        | Bắt buộc                   | `-t <threads>`          |
| 2    | MultiQC       | Tổng hợp báo cáo FastQC nhiều mẫu thành 1 báo cáo tổng thể     | Thư mục chứa kết quả FastQC    | `multiqc_report.html`, thư mục `multiqc_data/` | Bắt buộc | —           |
| 3    | fastp         | Tách adapter, cắt bỏ bases chất lượng thấp, lọc read ngắn     | R1.fastq, R2.fastq thô         | R1.clean.fastq, R2.clean.fastq + báo cáo HTML/JSON | Bắt buộc | `--detect_adapter_for_pe`, chất lượng cắt, chiều dài tối thiểu |
| 4    | FastQC       | Kiểm tra chất lượng dữ liệu sau khi lọc                        | Fastq đã lọc từ bước 3        | Báo cáo FastQC (HTML/ZIP)      | Bắt buộc                   | `-t <threads>`          |
| 5    | MultiQC      | Tổng hợp báo cáo FastQC của dữ liệu đã lọc                     | Thư mục FastQC (bước 4)        | `multiqc_report.html`          | Bắt buộc                   | —                       |
| 6    | SeqKit stats | Tính thống kê cơ bản của reads (số reads, độ dài)              | R1.clean.fastq, R2.clean.fastq | Bảng thống kê in ra màn hình   | Bắt buộc                   | —                       |
| 7    | SPAdes       | Lắp ráp de novo trình tự genome vi khuẩn                       | R1.clean.fastq, R2.clean.fastq | `contigs.fasta`, (có thể có `scaffolds.fasta`) và các file log | Bắt buộc | `--isolate`, `-t <threads>`, `-m <RAM>` |
| 8    | QUAST        | Đánh giá độ dài, N50, số contig, GC, lỗi lắp ráp của assembly   | `contigs.fasta` (có thể có tham chiếu) | Báo cáo QUAST (HTML, TSV, PDF)  | Bắt buộc                   | `-t <threads>`; `-r ref.fasta` (nếu có) |
| 9    | Barrnap      | Dò tìm gene RNA (5S, 16S, 23S rRNA) trong genome                | `contigs.fasta`                | GFF3 ghi tọa độ rRNA           | Bắt buộc                   | `--kingdom bac` (HMM của vi khuẩn) |
| 10   | grep + bedtools | Trích xuất trình tự 16S rRNA từ GFF (theo strand)           | `*.gff` từ Barrnap             | `16S.fasta` (trình tự 16S)     | Tùy điều kiện (nếu tìm thấy 16S) | `grep "16S_rRNA" rrna.gff`; `bedtools getfasta -fi contigs.fasta -bed 16S.gff -s -name` |
| 11   | SeqKit stats | Thống kê độ dài và số lượng các trình tự 16S (đa phần ~1500 bp) | `16S.fasta`                   | Thống kê in ra màn hình       | Tùy chọn (kiểm tra 16S)   | —                       |
| 12   | FastANI      | Tính độ tương đồng toàn bộ genome (ANI) với genome tham chiếu   | `contigs.fasta`, genome_ref.fasta | File kết quả ANI (tab-delimited) | Có tham chiếu             | `-q contigs.fasta -r ref.fasta -o out.ani` |
| 13   | Unicycler    | Lắp ráp thêm (thay thế SPAdes) với bộ đọc ngắn (short reads)   | R1.clean.fastq, R2.clean.fastq | FASTA assembly (thường `assembly.fasta`), log | Có điều kiện (nếu SPAdes kém) | `-1 reads_1.fastq -2 reads_2.fastq -o out/ -t <threads>` |
| 14   | QUAST        | Đánh giá chất lượng assembly từ Unicycler                       | Assembly Unicycler (contigs)   | Báo cáo QUAST mới             | Có điều kiện              | Như bước 8              |
| 15   | RagTag       | Sắp xếp (scaffold) contigs theo genome tham chiếu                | `contigs.fasta`, ref.fasta     | `ragtag.scaffold.fasta`, AGP    | Có điều kiện (nếu assembly phân mảnh & có ref.) | `scaffold ref.fa contigs.fa` |
| 16   | QUAST + SeqKit | Đánh giá scaffold (số contig, N50, GC, số ký tự N)          | `ragtag.scaffold.fasta`       | Báo cáo QUAST, bảng thống kê   | Có điều kiện              | Như bước 8, thêm đếm N: `grep -o "N" scaffold.fasta \| wc -l` |
| 17   | Bakta        | Chú giải bộ gen vi khuẩn: gene, CDS, rRNA, tRNA, AMR, VF, ...   | Assembly cuối cùng (contigs/scaffolds) | GFF3, GenBank, EMBL, FAA, FFN, TSV, JSON | Bắt buộc (bước kết thúc khi thành công) | `-i assembly.fasta -o bakta_out/ -t <threads> --prefix <sample>` |

*(Bước 13–16 có thể lặp lại tùy kết quả: sau Unicycler chọn assembly tốt nhất; nếu cần và có tham chiếu gần, chạy RagTag để xếp lại contigs. Bước 10–11 (16S) chỉ làm khi Barrnap tìm thấy gene 16S. FastANI yêu cầu có genome tham chiếu thuộc loài gần. Các tham số chính ví dụ như chỉ định number of threads (`-t`) hoặc chế độ (`--isolate`) đã được ghi chú.)*

## Mô tả chi tiết các bước

### Bước 1: Kiểm tra chất lượng dữ liệu thô (FastQC, MultiQC)

- **Công cụ:** [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) kiểm tra chất lượng reads (phân phối điểm chất lượng theo base, GC content, duplicated reads, adapter…); [MultiQC](https://github.com/ewels/MultiQC) tổng hợp kết quả của nhiều báo cáo.
- **Lệnh mẫu:** 
  ```bash
  fastqc RawRead/*.fastq.gz -o QC/FastQC_raw -t 8
  multiqc QC/FastQC_raw -o QC/MultiQC_raw
  ```
  (Thay `RawRead/*.fastq.gz` bằng đường dẫn thực tới file FASTQ thô. `-t` là số luồng CPU.)
- **Giải thích:** FastQC tạo báo cáo HTML/ZIP riêng cho mỗi file input. MultiQC quét thư mục kết quả FastQC và sinh một báo cáo tổng hợp duy nhất, giúp so sánh đồng thời nhiều mẫu hoặc nhiều luồng phân tích.
- **Đầu ra:** Các file `fastqc.zip` và `fastqc.html` trong `QC/FastQC_raw/`, và file `multiqc_report.html` trong `QC/MultiQC_raw/`. Kiểm tra các cảnh báo (warnings) như adapter còn sót, base chất lượng thấp, GC bất thường...

### Bước 2: Làm sạch reads (fastp)

- **Công cụ:** [fastp](https://github.com/opengene/fastp) – công cụ nhanh gọn cho cắt và lọc reads.
- **Lệnh mẫu:** 
  ```bash
  fastp \
    -i RawRead/sample_R1.fastq.gz \
    -I RawRead/sample_R2.fastq.gz \
    -o sample/sample_R1.clean.fastq \
    -O sample/sample_R2.clean.fastq \
    --detect_adapter_for_pe \
    --cut_right --cut_right_window_size 4 --cut_right_mean_quality 20 \
    --qualified_quality_phred 20 --unqualified_percent_limit 20 \
    --length_required 50 --correction \
    --thread 8 \
    --html sample/fastp_report.html \
    --json sample/fastp_report.json
  ```
  (Điền đúng tên file đầu vào / đầu ra tương ứng. Tùy chọn `--detect_adapter_for_pe` tự phát hiện và loại bỏ adapter trong dữ liệu paired-end. Các tham số `cut_right`, `qualified_quality_phred`, v.v. thiết lập ngưỡng chất lượng và chiều dài tối thiểu sau lọc.)
- **Giải thích:** fastp cắt adapter, loại bỏ bases có chất lượng thấp ở đầu/cuối read (sử dụng sliding window) và loại bỏ các read có nhiều base kém (theo `--qualified_quality_phred`, `--unqualified_percent_limit`). Tùy chọn `--correction` kích hoạt so khớp đoạn chồng chéo giữa hai đầu read để sửa lỗi. Fastp tạo báo cáo HTML/JSON thống kê số lượng reads giữ lại, phân phối chất lượng, tiết lộ adapter bị loại.
- **Đầu ra:** Hai file FASTQ sạch (`sample_R1.clean.fastq`, `sample_R2.clean.fastq`) trong thư mục mẫu; file `fastp_report.html` và `fastp_report.json` cho báo cáo. Nếu dung lượng bộ nhớ (RAM) thấp, có thể cân nhắc dùng `--thread` hợp lý.

### Bước 3: Kiểm tra dữ liệu sau khi làm sạch (FastQC, MultiQC, SeqKit)

- **Công cụ:** FastQC/MultiQC như ở Bước 1, và [SeqKit](https://bioinf.shenwei.me/seqkit/) để thống kê reads.
- **Lệnh mẫu:** 
  ```bash
  fastqc sample/*.clean.fastq -o QC/FastQC_clean -t 8
  multiqc QC/FastQC_clean -o QC/MultiQC_clean
  seqkit stats sample/sample_R1.clean.fastq sample/sample_R2.clean.fastq
  ```
- **Giải thích:** Chạy FastQC và MultiQC lại trên dữ liệu sạch để chắc rằng chất lượng đọc đã được cải thiện (ít lỗi, ít adapter hơn). SeqKit `stats` tính số lượng reads, tổng ký tự và độ dài ngắn/dài nhất, kiểm tra có read bị ngắn bất thường. Nếu thấy vấn đề (ví dụ quá ít reads, phân phối chiều dài lạ), cần điều chỉnh tham số fastp hoặc kiểm tra lỗi kỹ hơn.
- **Đầu ra:** Báo cáo FastQC/MultiQC trong `QC/FastQC_clean` và `QC/MultiQC_clean`; bảng thông số SeqKit in ra terminal.

### Bước 4: Lắp ráp de novo (SPAdes)

- **Công cụ:** [SPAdes](https://cab.spbu.ru/software/spades/) – bộ lắp ráp genome thế hệ mới.
- **Lệnh mẫu:** 
  ```bash
  spades.py \
    --isolate \
    -1 sample/sample_R1.clean.fastq \
    -2 sample/sample_R2.clean.fastq \
    -o sample/spades_output \
    -t 8 -m 16
  ```
- **Giải thích:** Tham số `--isolate` cho SPAdes biết đây là dữ liệu bộ gen vi khuẩn thông thường. Các tham số `-t` (số luồng) và `-m` (GB RAM) được điều chỉnh tùy phần cứng. SPAdes đọc input và tạo ra nhiều contigs. Kết quả lưu trong thư mục `sample/spades_output`, bao gồm `contigs.fasta` (các contig ghép được) và thường có `scaffolds.fasta` (nếu SPAdes có thể ghép thêm).
- **Đầu ra:** File `contigs.fasta` (các contig lắp ráp được), file `scaffolds.fasta`, file đồ thị lắp ráp (`assembly_graph.fastg`) và các log. Nếu SPAdes không hoàn thành (lỗi phân mảnh quá nhiều, lỗi tĩnh bộ nhớ...), cần xem lại dữ liệu đầu vào hoặc thử các công cụ khác (như Unicycler).

### Bước 5: Đánh giá chất lượng lắp ráp (QUAST)

- **Công cụ:** [QUAST](http://quast.sourceforge.net/) – công cụ đánh giá chất lượng assembly.
- **Lệnh mẫu:** 
  ```bash
  quast.py sample/spades_output/contigs.fasta \
    -o sample/quast_spades -t 8
  ```
  (Nếu có genome tham chiếu, thêm `-r ref_genome.fasta` để tính thêm các chỉ số so với tham chiếu.)
- **Giải thích:** QUAST tính các chỉ số thống kê: tổng kích thước assembly, số contig, N50, độ lệch so với tham chiếu nếu có, GC%, số misassemblies... QUAST cho phép đánh giá xem assembly có đủ tốt hay không. QUAST làm việc được cả khi không có file tham chiếu.
- **Đầu ra:** Thư mục `sample/quast_spades/` chứa báo cáo (HTML, TSV, PDF) với các số liệu. Kiểm tra xem tổng độ dài có hợp lý, N50 cao, số contig tối ưu, không nhiều misassembly.
  
### Bước 6: Tìm gene rRNA (Barrnap)

- **Công cụ:** [Barrnap](https://github.com/tseemann/barrnap) – công cụ tìm các gene RNA (rRNA, tRNA, ...).
- **Lệnh mẫu:** 
  ```bash
  barrnap --kingdom bac sample/spades_output/contigs.fasta > sample/rrna.gff
  ```
- **Giải thích:** Barrnap sử dụng các mô hình HMM để xác định vị trí gene rRNA (5S, 16S, 23S) trong bộ gen vi khuẩn. Kết quả được ghi vào file GFF3 (`rrna.gff`). Mỗi dòng GFF sẽ chứa loại `rRNA` và tên (`Name=16S_rRNA` chẳng hạn) và vị trí bắt đầu/kết thúc.
- **Đầu ra:** File `rrna.gff` trong thư mục mẫu. Dùng `grep "16S"` để kiểm tra xem có phát hiện 16S rRNA hay không.

### Bước 7: Trích xuất trình tự 16S rRNA (grep + bedtools)

- **Công cụ:** [grep](https://en.wikipedia.org/wiki/Grep) (lệnh hệ thống) và [BEDTools](https://bedtools.readthedocs.io/en/latest/content/tools/getfasta.html).
- **Lệnh mẫu:** 
  ```bash
  grep "16S_rRNA" sample/rrna.gff > sample/16S.gff
  bedtools getfasta -fi sample/spades_output/contigs.fasta \
    -bed sample/16S.gff -s -name \
    > sample/16S.fasta
  ```
- **Giải thích:** Dòng `grep` lọc ra các dòng GFF của Barrnap chứa chuỗi `16S_rRNA`. Sau đó, `bedtools getfasta` lấy các đoạn trình tự tương ứng từ file FASTA của assembly theo tọa độ trong GFF. Tùy chọn `-s` đảm bảo giữ chiều mạch (nếu gene nằm trên mạch âm thì sẽ lấy bổ sung bội); `-name` dùng trường tên trong GFF làm tiêu đề FASTA. Kết quả thu được là file FASTA của các gene 16S rRNA (nếu có).
- **Đầu ra:** File `16S.fasta` chứa trình tự 16S rRNA (đa phần dài ~1500 bp cho vi khuẩn). Nếu không tìm thấy 16S, có thể do assembly phân mảnh hoặc gene bị cắt ngắn; trong trường hợp này, có thể kiểm tra ở assembly thay thế (Unicycler) hoặc tìm kiếm khác.

### Bước 8: Thống kê 16S rRNA (SeqKit stats)

- **Công cụ:** SeqKit.
- **Lệnh mẫu:** 
  ```bash
  seqkit stats sample/16S.fasta
  ```
- **Giải thích:** Công cụ này in ra số reads (số gene 16S tìm được), tổng chiều dài, độ dài trung bình, v.v. Thường kỳ vọng 1–2 gene 16S đầy đủ (~1500 bp) nếu có ít nhất một operon 16S-23S-tRNAs.
- **Đầu ra:** Bảng thống kê in ra terminal, cho biết đã tìm được bao nhiêu gene 16S và độ dài của chúng. Dùng để kiểm tra xem Barrnap đã tìm được gene có đủ dài hay chỉ partial (nhỏ hơn ~1400 bp).

### Bước 9: So sánh toàn bộ genome (FastANI)

- **Công cụ:** [FastANI](https://github.com/ParBLiSS/FastANI) – tính toán nhanh chỉ số Average Nucleotide Identity (ANI).
- **Lệnh mẫu:** 
  ```bash
  fastANI -q sample/spades_output/contigs.fasta \
    -r ref/Bacillus_velezensis_FZB42.fasta \
    -o sample/ani_results.txt
  ```
- **Giải thích:** FastANI đo lường độ tương đồng nucleotide trung bình giữa hai genome. Giá trị ANI trên 95% thường cho biết hai genome cùng loài. Công cụ này không cần căn chỉnh dọc chuỗi đầy đủ mà dùng phương pháp MinHash, chạy rất nhanh. Kết quả (`ani_results.txt`) có dạng tab: `query_genome  reference_genome  ANI%  bidirectional_mappings  total_fragments`.
- **Đầu ra:** File `ani_results.txt`. Kiểm tra giá trị ANI và tỉ lệ mapping để xác nhận loài tham chiếu phù hợp (nên trên ~90–95% cho cùng loài).

### Bước 10: Lắp ráp thay thế (Unicycler) *[tùy chọn]*

- **Công cụ:** [Unicycler](https://github.com/rrwick/Unicycler) – pipeline lắp ráp tập trung cho vi khuẩn (hỗ trợ tập short-reads và hybrid long-reads).
- **Khi thực hiện:** Nếu kết quả lắp ráp SPAdes ở Bước 7 không đạt yêu cầu (quá nhiều contigs, N50 thấp, hoặc nhiều lỗ hổng), có thể chạy Unicycler như một phương án thay thế với cùng dữ liệu short reads.
- **Lệnh mẫu:** 
  ```bash
  unicycler -1 sample_R1.clean.fastq \
            -2 sample_R2.clean.fastq \
            -o sample/unicycler_out \
            -t 8
  ```
- **Giải thích:** Unicycler sẽ tự động sử dụng SPAdes ở chế độ short-read rồi xử lý thêm để tối ưu lắp ráp. Tùy chọn `-t` số luồng CPU. (Nếu có long reads, có thể thêm tham số `-l longreads.fastq` để chạy hybrid, nhưng trong quy trình này giả định chỉ có short reads.)
- **Đầu ra:** Thư mục `sample/unicycler_out/` bao gồm file `assembly.fasta` (các contig/scaffold cuối cùng của Unicycler), cùng file log. Tiêu đề mặc định có thể là `contigs.fasta` hoặc `assembly.fasta` tùy phiên bản. Nếu Unicycler thành công, nên kiểm tra lại chất lượng tương tự như với SPAdes.

### Bước 11: Đánh giá assembly Unicycler (QUAST)

- **Công cụ:** QUAST (tương tự Bước 5).
- **Lệnh mẫu:** 
  ```bash
  quast.py sample/unicycler_out/assembly.fasta \
    -o sample/quast_unicycler -t 8
  ```
- **Giải thích:** Tương tự đánh giá assembly SPAdes. So sánh các chỉ số (số contig, N50, lỗi) giữa SPAdes và Unicycler. Lựa chọn assembly tốt hơn làm base cho bước kế tiếp. Nếu Unicycler tốt hơn (few contigs, N50 cao hơn, ít lỗi), chuyển sang bước chú giải với assembly này. Nếu không, vẫn có thể thử RagTag với assembly cũ.

### Bước 12: Scaffold theo tham chiếu (RagTag) *[tùy chọn]*

- **Công cụ:** [RagTag](https://github.com/malonge/RagTag) – sắp xếp và nối contig theo genome tham chiếu.
- **Khi thực hiện:** Nếu assembly (SPAdes hoặc Unicycler) vẫn còn phân mảnh và bạn có một genome tham chiếu cùng loài hoặc rất gần, có thể dùng RagTag để ghép contigs dựa trên homology.
- **Lệnh mẫu:** 
  ```bash
  ragtag.py scaffold ref/Bacillus_velezensis_FZB42.fasta \
                   sample/spades_output/contigs.fasta \
                   -o sample/ragtag_out -t 8
  ```
- **Giải thích:** RagTag sẽ căn cứ vào genome tham chiếu để sắp xếp, định hướng và nối các contigs của bạn. Kết quả là `ragtag.scaffold.fasta` và file AGP (với thông tin gap). Tùy chọn `-t` số luồng. (Nếu đã dùng Unicycler, đổi `contigs.fasta` thành `unicycler_out/assembly.fasta`.) Nếu ANI ở Bước 9 quá thấp (<90%), tránh dùng RagTag để không ép lắp ráp sai.
- **Đầu ra:** Thư mục `sample/ragtag_out/` chứa `ragtag.scaffold.fasta`, file `ragtag.agp` (mô tả vị trí các gap), và log. Đây là bản assembly cuối cùng (nếu được chấp nhận).
- **Chú ý:** RagTag không khắc phục lỗi từ chối hoàn toàn, nhưng sắp xếp contigs tốt hơn nếu có reference phù hợp. Kiểm tra kỹ scaffold mới để tránh misassembly do reference không quá gần.

### Bước 13: Đánh giá scaffold (QUAST, SeqKit)

- **Công cụ:** QUAST, SeqKit.
- **Lệnh mẫu:** 
  ```bash
  quast.py sample/ragtag_out/ragtag.scaffold.fasta \
    -r ref/Bacillus_velezensis_FZB42.fasta \
    -o sample/quast_ragtag -t 8
  seqkit fx2tab -n -l -g sample/ragtag_out/ragtag.scaffold.fasta
  grep -o "N" sample/ragtag_out/ragtag.scaffold.fasta | wc -l
  ```
- **Giải thích:** Đánh giá chất lượng bản scaffold tương tự. Kiểm tra N50, L50, số chuỗi dài. Dùng `seqkit fx2tab` để liệt kê tên và độ dài mỗi contig (đối chiếu với contigs cũ). Đếm số ký tự `N` (thay thế gap) để biết đã tạo bao nhiêu khoảng trống (ký tự `N`) trong scaffold. So sánh với bản assembly chưa scaffold để chắc chắn cải thiện.
- **Đầu ra:** Báo cáo QUAST (tương tự) trong `sample/quast_ragtag/`, bảng thống kê từ `seqkit`, và giá trị `N` từ lệnh `grep`. Chọn bản assembly (contigs hoặc scaffold) có chất lượng cao hơn để bước chú giải cuối cùng.

### Bước 14: Chú giải genome (Bakta)

- **Công cụ:** [Bakta](https://bakta.readthedocs.io/) – chú giải bộ gen vi khuẩn.
- **Lệnh mẫu:** 
  ```bash
  bakta \
    --input sample/final_assembly.fasta \
    --output sample/bakta_output \
    --prefix SampleID \
    --threads 8
  ```
  (Thay `final_assembly.fasta` bằng file contigs hoặc scaffold cuối cùng bạn chọn; `SampleID` là tiền tố đẳng dạng cho các file đầu ra.)
- **Giải thích:** Bakta thực hiện chú giải toàn diện: tìm CDS, gene RNA (rRNA, tRNA), CRISPR, chức năng protein, liên kết thư viện DBxref,… Kết quả đầy đủ trong nhiều định dạng (GFF3, GenBank, FASTA protein/nt, bảng TSV báo cáo). Bakta thiết kế cho vi khuẩn và sử dụng phương pháp tìm kiếm không căn chỉnh nhanh, thường hoàn tất một genome ~5 Mb trong vài phút. Nên kiểm tra log của Bakta để biết các bản ghi (annotation) và phiên bản cơ sở dữ liệu đã dùng.
- **Đầu ra:** Thư mục `sample/bakta_output/` chứa nhiều file:
  - `SampleID.gff` (chú giải GFF3 chuẩn GenBank),
  - `SampleID.gbk` (GenBank), `SampleID.embl` (EMBL),
  - `SampleID.ffn` (nucleotides of CDS), `SampleID.faa` (proteins),
  - `SampleID.tsv` (tóm tắt chức năng), `SampleID.json`, v.v.
- Sau bước này, quy trình kết thúc **thành công**. Kết quả chú giải có thể được dùng để nộp lên public database hoặc phân tích sau.

## Tiêu chí hoàn thành và điều kiện thực hiện

- **Bước cuối thành công:** Quy trình kết thúc khi Bakta hoàn tất chú giải genome của mẫu đã chọn (contigs hoặc scaffolds cuối cùng). Lấy kết quả từ thư mục `bakta_output/` làm output cuối.
- **Kiểm tra assembly:** Sau mỗi assembly (SPAdes, Unicycler, RagTag), luôn đánh giá với QUAST. Nếu các chỉ số (ví dụ số contig quá nhiều, N50 thấp, tổng kích thước bất thường, misassemblies) không chấp nhận được, chuyển sang giải pháp thay thế.
  - Nếu SPAdes đạt yêu cầu ngay: bỏ qua Unicycler/RagTag và tiếp tục.
  - Nếu SPAdes kém: chạy Unicycler để so sánh (thường làm sớm vì Unicycler có thể cho assembly tốt hơn với short-reads).
  - Nếu vẫn phân mảnh & có reference phù hợp: chạy RagTag.
- **Tham chiếu (FastANI, RagTag):** Nếu FastANI (Bước 12) cho giá trị thấp (<90-95%), có thể genome không cùng loài – khi đó cần tìm reference khác hoặc thôi không dùng RagTag. Chỉ dùng RagTag nếu chắc chắn reference gần.
- **16S rRNA:** Nếu Barrnap không tìm thấy 16S (vì assembly thiếu vùng rRNA), có thể thử ở assembly khác (Unicycler) hoặc kiểm tra why không lắp ráp được (16S thường lặp). Trường hợp không tìm thấy 16S ở cả hai assembly có thể do phân mảnh cao.
- **Phân chia bước:** Trong bảng trên, bước nào đánh dấu “bắt buộc” luôn chạy; “có điều kiện” chỉ chạy nếu cần. Ví dụ Unicycler (Bước 13 trong bảng) chỉ thực hiện nếu SPAdes thất bại; RagTag (Bước 15-16) chỉ khi cần cải thiện assembly với tham chiếu.

> **Lưu ý:** Trình tự và cách chạy có thể điều chỉnh tùy theo dữ liệu và mục tiêu. Ví dụ, có thể chạy Unicycler song song để so sánh, nhưng chỉ dùng kết quả tốt nhất. Luôn sao lưu dữ liệu quan trọng và kiểm tra kỹ báo cáo tại mỗi bước để đảm bảo quy trình diễn ra đúng. 

---

## Summary Table (English)

| Step | Tool     | Purpose                                      | Inputs                    | Outputs                             | Mandatory/Optional    | Key parameters                |
|------|----------|----------------------------------------------|---------------------------|-------------------------------------|-----------------------|-------------------------------|
| 1    | FastQC   | Check raw read quality                       | Raw FASTQ files           | FastQC HTML/ZIP reports             | Mandatory             | `-t threads`                  |
| 2    | MultiQC  | Aggregate FastQC results                     | Folder of FastQC results  | `multiqc_report.html`, `multiqc_data/` | Mandatory           | —                             |
| 3    | fastp    | Trim adapters, filter low-quality reads      | Raw R1.fastq, R2.fastq     | Cleaned FASTQ files + HTML/JSON report | Mandatory         | `--detect_adapter_for_pe`, quality and length filters |
| 4    | FastQC   | Check trimmed-read quality                   | Cleaned FASTQ files       | FastQC reports                      | Mandatory             | `-t threads`                  |
| 5    | MultiQC  | Aggregate post-filter FastQC results         | Folder of FastQC results  | `multiqc_report.html`               | Mandatory             | —                             |
| 6    | SeqKit   | Compute read statistics (count, length)      | Cleaned FASTQ files       | Stats table on stdout               | Mandatory             | —                             |
| 7    | SPAdes   | De novo genome assembly                      | Cleaned R1.fastq, R2.fastq | `contigs.fasta` (assembly)          | Mandatory             | `--isolate`, `-t threads`, `-m memory` |
| 8    | QUAST    | Evaluate assembly quality                    | `contigs.fasta` (+ ref)    | Assembly quality reports            | Mandatory             | `-t threads`, `-r ref.fasta` |
| 9    | Barrnap  | Find rRNA genes (5S, 16S, 23S)               | `contigs.fasta`           | GFF3 with rRNA coordinates          | Mandatory             | `--kingdom bac`     |
| 10   | grep + bedtools | Extract 16S rRNA sequences from GFF   | `rrna.gff`                 | `16S.fasta`                         | Optional (if 16S found) | `grep "16S_rRNA"`, `bedtools getfasta -s -name` |
| 11   | SeqKit   | Stats on 16S sequences                      | `16S.fasta`               | Stats table on stdout               | Optional             | —                             |
| 12   | FastANI  | Compute genome-wide ANI vs reference         | Assembly FASTA, ref FASTA  | ANI results (tab file)             | Conditional (if ref.) | `-q assembly -r reference -o output` |
| 13   | Unicycler| Alternative assembly (short-read only)       | Cleaned R1.fastq, R2.fastq | `assembly.fasta` (Unicycler output) | Optional             | `-1 reads -2 reads -o out -t threads` |
| 14   | QUAST    | Evaluate Unicycler assembly                  | Unicycler `assembly.fasta` | Assembly reports                    | Optional             | Same as Step 8                |
| 15   | RagTag   | Reference-guided scaffolding of contigs      | Assembly FASTA, ref FASTA  | `ragtag.scaffold.fasta`             | Conditional (if needed) | `ragtag.py scaffold ref.fa contigs.fa` |
| 16   | QUAST + SeqKit | Evaluate scaffolded assembly             | `ragtag.scaffold.fasta`   | Reports; contig stats (SeqKit)      | Conditional           | Same as Step 8 + count `N`    |
| 17   | Bakta    | Annotate final genome (genes, proteins, etc.)| Final assembly FASTA       | GFF3, GenBank, FASTA, TSV, JSON      | Mandatory (final step) | `--input assembly.fa --output dir --prefix ID` |

This table summarizes each step: **Tool**, **Purpose**, **Inputs/Outputs**, **When to run** (mandatory or conditional), and **Key parameters**. (Steps like Unicycler and RagTag are optional alternatives run only if needed. FastANI requires a reference genome.) The final step is always running Bakta on the chosen assembly to complete annotation.
