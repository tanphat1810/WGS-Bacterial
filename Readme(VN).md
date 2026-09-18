# Quy trình lắp ráp và chú giải bộ gen vi khuẩn

**I. Tóm tắt:** 
<div align="justify">
Đây là quy trình phân tích dữ liệu Whole Genome Sequencing (WGS) của mẫu vi khuẩn. Các bước chính bao gồm: kiểm tra chất lượng dữ liệu đầu vào, làm sạch adapter và lọc reads, thực hiện lắp ráp de novo, đánh giá chất lượng lắp ráp, phát hiện gen rRNA (bao gồm 16S) và trích xuất trình tự tương ứng, so sánh độ tương đồng genome (ANI) với genome tham chiếu để định danh loài, và cuối cùng là chú giải genome bằng công cụ Bakta. Hình dưới đây minh họa tổng quát luồng công việc; bảng tóm tắt các bước được trình bày bên dưới; và các phần chi tiết mô tả từng công cụ, lệnh, đầu vào/đầu ra, cùng các lưu ý thực thi được giải thích bên dưới.
</div>

![Quy trình lắp ráp và chú giải bộ gen](QUY%20TRÌNH%20LẮP%20RÁP%20VÀ%20CHÚ%20GIẢI%20BỘ%20GEN%20VI%20KHUẨN.svg) 

## II. Chuẩn bị môi trường phần mềm

Quy trình phân tích yêu cầu một môi trường conda để cài đặt các công cụ bioinformatics phổ biến

```bash
conda create -n <env> -c conda-forge -c bioconda <Tool>
conda activate <env>
```
<div align="justify">
  
*(Các phiên bản cụ thể của từng công cụ có thể được chỉ định, tùy ý. Một số công cụ cần database như Bakta nếu lần đầu sử dụng, có thể cần chạy bakta_db install để tải database; một số công cụ Barrnap đã có sẵn các mô hình HMM rRNA sau khi cài thì không cần tự cài, trừ trường hợp tự setup database mong muốn).*.  
</div>

## Cấu trúc thư mục

Ví dụ về cấu trúc thư mục cho quy trình này:

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

## IV. Bảng tóm tắt các công cụ

| Stt  | Công cụ         | Mục đích                                                    
|:----:|:----------------|:----------------------------------------------------------------|
| 1    | FastQC          | Kiểm tra chất lượng  reads                                      |
| 2    | MultiQC         | Tổng hợp báo cáo                                                |
| 3    | Fastp           | Loại bỏ bases chất lượng thấp, read ngắn, adapter               |
| 4    | SeqKit stats    | Thống kê chỉ số cơ bản của reads (số reads, độ dài)             |
| 5    | SPAdes          | Lắp ráp de novo trình tự genome vi khuẩn                        |
| 6    | QUAST           | Đánh giá độ dài, N50, số contig, GC, lỗi assembly               |
| 7    | Barrnap         | Dò tìm gene RNA (5S, 16S, 23S rRNA)                             |
| 8    | grep + bedtools | Trích xuất trình tự 16S rRNA từ GFF                             |
| 9    | FastANI         | Tính độ tương đồng genome với genome tham chiếu                 |
| 10   | MEGA12          | Xây dựng cây di truyền                                          |
| 11   | Unicycler       | Assembly cải thiện với bộ đọc ngắn (short reads)                |
| 12   | RagTag          | Sắp xếp (scaffold) contigs theo genome tham chiếu               |
| 13   | Bakta           | Chú giải bộ gen vi khuẩn: gene, CDS, rRNA, tRNA, AMR, VF, ...   |

## V. Mô tả chi tiết quy trình xử lý

### 5.1. Kiểm tra chất lượng dữ liệu thô (FastQC, MultiQC)

- [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) FastQC kiểm tra chất lượng reads (phân phối điểm chất lượng theo base, GC content, duplicated reads, adapter…); MultiQC tổng hợp kết quả báo cáo.
  
```bash
Fastqc <path/to/input.fastq.gz> -o <path/to/output_dir> -t 8
multiqc <path_to_fastqc_results> -o <path_to_multiqc_report>
```
(Thay `<path/to/input.fastq.gz>` bằng đường dẫn thực tới file FASTQ thô. `-t` là số luồng CPU.)

<div align="justify">

* Dữ liệu đầu vào `[Đường dẫn file fastq đầu vào]` của FastQC là các tệp FASTQ thô ở định dạng `.fastq` hoặc nén `.fastq.gz`. Công cụ này sẽ khởi tạo các file báo cáo `fastqc.zip` và `fastqc.html` riêng biệt cho từng tệp dữ liệu `.fastq` trong thư mục đầu ra được chỉ định. Đồng thời để công cụ chạy nhanh hơn có thể tăng số luồng bằng tham số `-t`. Sau đó, MultiQC nhận đầu vào `[Đường dẫn thư mục đầu vào]` là thư mục chứa đầu ra của FastQC và tạo một báo cáo tổng hợp duy nhất `report.html` và lưu trong thư mục đầu ra được chỉ định. Đầu ra giúp kiểm tra các cảnh báo (warnings) như adapter còn sót, base chất lượng thấp, GC bất thường...

</div>

### 5.2. Làm sạch reads (fastp)


- [fastp](https://github.com/opengene/fastp) – công cụ cắt và lọc reads.
- **Single-End**
- 
```bash
fastp \
  -i <path/to/input.fastq> \
  -o <path/to/output.fastq> \
  --cut_right --cut_right_window_size 4 --cut_right_window_size \
  --qualified_quality_phred 20 --unqualified_quality_phred \
  --length_required 50 \
  --thread 8 \
  --html <path/to/output.html> \
  --json <path/to/output.json>
```
  
- **Pair-end**

```bash
fastp \
  -i <path/to/inputR1.fastq> \
  -I <path/to/inputR2.fastq> \
  -o <path/to/outputR1.fastq> \
  -O <path/to/outputR2.fastq> \
  --detect_adapter_for_pe \
  --cut_right --cut_right_window_size 4 --cut_right_mean_quality 20 \
  --qualified_quality_phred 20 --unqualified_percent_limit 20 \
  --length_required 50 --correction \
  --thread 8 \
  --html <path/to/output.html> \
  --json <path/to/output.json>
```

<div align="justify">
  
Công cụ có thể chạy cả ở dạng Single-End và Pair-End. Single end là phương pháp giải trình tự mà máy chỉ đọc các base (nucleotide) từ một đầu duy nhất của đoạn DNA mục tiêu (thường bắt đầu từ đầu 5' và chạy theo một chiều). Kết quả tạo ra một đoạn đọc đơn (Single Read) cho mỗi mảnh DNA. Paired-end (PE) Sequencing (Giải trình tự đọc cặp): Là phương pháp giải trình tự mà máy sẽ đọc các base từ cả hai đầu (đầu 5' và đầu 3') của cùng một đoạn DNA mục tiêu. Quá trình này tạo ra hai đoạn đọc riêng biệt (được gọi là Forward Read/Read 1 và Reverse Read/Read 2). Tuy nhiên ở quy trình này chủ yếu tập trung pair end do độ chính xác cao.

</div>
 
Đường dẫn đầu vào được chỉ định bao gồm 2 file là R1 (cho file forward) và R2 (cho file Reverse), các tham số bao gồm:

> * **-i**: Đường dẫn đến file chứa các đoạn đọc xuôi (R1) đầu vào (định dạng `.fastq` hoặc `.fastq.gz`). 
> * **-I**: Đường dẫn đến file chứa các đoạn đọc ngược (R2) đầu vào.
> * **-o**: Đường dẫn để xuất file đầu ra từ R1 sau khi đã lọc và cắt.
> * **-O**: Đường dẫn để xuất file đầu ra từ R2 sau khi đã lọc và cắt. *(Sửa lỗi R1 thành R2)*
> * **--detect_adapter_for_pe**: Bật tính năng tự động phát hiện trình tự adapter.
> * **--cut_right**: Sẽ quét bắt đầu từ 5' đến đầu 3' (từ trái sang phải).
> * **--cut_right_window_size**: Đặt kích thước khi quét.
> * **--cut_right_mean_quality**: Ngưỡng chất lượng trung bình của cửa sổ.
> * **--qualified_quality_phred**: Quy định một base được coi là "đạt chất lượng" (qualified) nếu chất lượng của toàn read đó lớn hơn hoặc bằng ngưỡng.
> * **--unqualified_percent_limit**: Giới hạn tỷ lệ base kém chất lượng tối đa cho phép trong một read. Thông số đặt sẽ chuyển thành dạng phần trăm (%). Nếu một read chứa số base có chất lượng thấp hơn ngưỡng trên, toàn bộ read đó sẽ bị loại bỏ.
> * **--length_required**: Bộ lọc chiều dài tối thiểu, loại bỏ những reads có chiều dài thấp hơn ngưỡng.
> * **--correction**: Bật tính năng tự động sửa lỗi base (base correction) cho dữ liệu paired-end. Thuật toán sẽ tìm vùng chồng lấn giữa Read 1 và Read 2, nếu có sự sai khác về base ở vùng này, base có chất lượng cao hơn sẽ được dùng để sửa cho base có chất lượng thấp hơn.
> * **--thread**: Chọn số luồng (threads) của CPU để xử lý song song, giúp tăng tốc độ chạy lệnh.
> * **--html**: Đường dẫn xuất báo cáo chất lượng định dạng HTML (có thể mở bằng trình duyệt web để xem biểu đồ trực quan).
> * **--json**: Đường dẫn xuất báo cáo định dạng JSON (dùng để lưu trữ dữ liệu thô của báo cáo, thuận tiện cho các script lập trình xử lý tiếp).



### 5.3. Kiểm tra dữ liệu sau khi làm sạch (FastQC, MultiQC, SeqKit)

Sử dụng FastQC/MultiQC như ở Bước 1, và SeqKit để thống kê reads.

```bash
seqkit stats <path/to/inputR1.fastq> <path/to/inputR2.fastq> -o <path/to/input.txt>
```

<div align="justify">
FastQC và MultiQC kiểm tra dữ liệu sạch để chắc rằng chất lượng đọc đã được cải thiện (ít lỗi, ít adapter hơn). Đầu vào của Seqkit sẽ lần lượt là 2 file fastq R1 và R2 đã được lọc. Tham số stats trong Seqkit tính số lượng reads, tổng ký tự và độ dài ngắn/dài nhất, kiểm tra có read bị ngắn bất thường. Nếu thấy vấn đề (ví dụ quá ít reads, phân phối chiều dài lạ), cần điều chỉnh tham số fastp hoặc kiểm tra lỗi kỹ hơn.
</div>

### 5.4. Lắp ráp de novo (SPAdes)

- [SPAdes](https://cab.spbu.ru/software/spades/) – bộ lắp ráp genome
  
  ```bash
  spades.py \
  --isolate \
  -1 <path/to/inputR1.fastq> \
  -2 <path/to/inputR2.fastq> \
  -o <path/to/output_dir> \
  -t 8 -m 16
  ```

<div align="justify">
Đầu vào của SPAdes sẽ lần lượt là 2 file fastq R1 và R2 đã được lọc. Tham số --isolate cho SPAdes biết đây là dữ liệu bộ gen vi khuẩn thông thường. Các tham số -t (số luồng) và -m (GB RAM) được điều chỉnh tùy phần cứng. SPAdes đọc input và tạo ra nhiều contigs. Kết quả lưu trong thư mục đầu ra chỉ định, bao gồm contigs.fasta (các contig ghép được) và thường có scaffolds.fasta (nếu SPAdes có thể ghép thêm).
</div>

### 5.5. Đánh giá chất lượng lắp ráp (QUAST)

- [QUAST](http://quast.sourceforge.net/) – công cụ đánh giá chất lượng assembly.

  ```bash
  quast.py <path/to/input.fasta> \
    -o <path/to/output_dir> -t 8
  ```
<div align="justify">
(Nếu có genome tham chiếu, thêm -r ref_genome.fasta để tính thêm các chỉ số so với tham chiếu).
  
QUAST nhận đầu vào là file fasta đầu ra của SPAdes và tính các chỉ số thống kê: tổng kích thước assembly, số contig, N50, độ lệch so với tham chiếu nếu có, GC%, số misassemblies... QUAST cho phép đánh giá xem assembly có đủ tốt hay không. QUAST làm việc được cả khi không có file tham chiếu.
</div>
  
### 5.6. Tìm gene rRNA (Barrnap)

- [Barrnap](https://github.com/tseemann/barrnap) – công cụ tìm các gene RNA (rRNA, tRNA, ...).

  ```bash
  barrnap --kingdom bac <path/to/input.fasta> > <path/to/output.gff>
  ```

<div align="justify">
Barrnap sử dụng các mô hình HMM để xác định vị trí gene rRNA (5S, 16S, 23S) trong bộ gen vi khuẩn. --kingdom bac Chỉ định kingdom của sinh vật mục tiêu để công cụ áp dụng mô hình huấn luyện phù hợp. Trong trường hợp này, bac nghĩa là Bacteria (Vi khuẩn). Lệnh này sẽ tìm kiếm các đoạn rRNA 5S, 16S và 23S của vi khuẩn.(Các tùy chọn khác có thể dùng là arc cho Archaea, euk cho Eukaryota, và mito cho Ty thể động vật). Kết quả được ghi vào file GFF3. Mỗi dòng GFF sẽ chứa loại rRNA, tên và vị trí bắt đầu/kết thúc.
</div>

### 5.7. Trích xuất trình tự 16S rRNA (grep + bedtools)

- [grep](https://en.wikipedia.org/wiki/Grep) (lệnh hệ thống) và [BEDTools](https://bedtools.readthedocs.io/en/latest/content/tools/getfasta.html).

  ```bash
  grep "16S_rRNA" <path/to/input.gff> > <path/to/output16S.gff>
  bedtools getfasta -fi <path/to/input.fasta> \
                    -bed <path/to/input16S.gff> -s -name \
                    > <path/to/output16S.fasta>
  ```

<div align="justify">
Ở lệnh grep "16S_rRNA" là từ khóa cần tìm kiếm. Lệnh sẽ quét qua toàn bộ file và chỉ giữ lại các dòng có chứa chính xác cụm từ này. File đầu vào (chứa tất cả các loại rRNA như 5S, 16S, 23S). File đầu ra mới, lúc này chỉ còn chứa tọa độ vị trí của các gen 16S rRNA. Sau đó, lệnh bedtools getfasta lấy đầu vào là file fasta chứa toàn bộ trình tự các đoạn contigs (được tạo ra từ phần mềm lắp ráp SPAdes). -bed sample/16S.gff chỉ định file chứa tọa độ vùng cần cắt (Lấy từ file đầu ra ở mục 5.6). Lệnh sẽ dựa vào đây để biết gen 16S nằm ở contig nào, từ nucleotide thứ bao nhiêu đến thứ bao nhiêu. -s viết tắt của strand-specific giúp bedtools tự động lấy mạch bổ sung ngược (reverse complement) nếu gen 16S đó nằm trên mạch trừ (-) của DNA. -name sử dụng chính tên của gen (hoặc ID) có trong file GFF để đặt tên cho các header của chuỗi fasta đầu ra, giúp dễ dàng nhận biết đoạn trình tự đó thuộc về gen nào thay vì chỉ hiển thị tọa độ số chung chung.
</div>

### 5.8. Thống kê 16S rRNA (SeqKit stats)

  ```bash
  seqkit stats <path/to/input16s.fasta>
  ```
Công cụ này in ra số reads (số gene 16S tìm được), tổng chiều dài, độ dài trung bình, v.v. Thường kỳ vọng 1–2 gene 16S đầy đủ (~1500 bp) nếu có ít nhất một operon 16S-23S-tRNAs.

### 5.9. So sánh toàn bộ genome (FastANI)

- [FastANI](https://github.com/ParBLiSS/FastANI) – tính toán chỉ số Average Nucleotide Identity (ANI).

  ```bash
  fastANI -q <path/to/input.fasta> \
          -r <path/to/input_ref.fasta> \
          -o <path/to/output.txt>
  ```

<div align="justify">
FastANI nhận đầu vào file chứa contig và bộ genome tham chiếu để đo lường độ tương đồng nucleotide trung bình giữa hai genome. Giá trị ANI trên 95% thường cho biết hai genome cùng loài. Công cụ này không cần căn chỉnh dọc chuỗi đầy đủ mà dùng phương pháp MinHash, chạy rất nhanh. Kết quả có dạng tab: query_genome  reference_genome  ANI%  bidirectional_mappings  total_fragments.
</div>

### 5.10. Xây dựng cây di truyền (MEGA12)
<div align="justify">
Công cụ được xử dụng là MEGA12 ở dạng desktop lấy đầu vào là file fasta 16s được lắp ráp và các fasta 16s của các loài cùng chi hoặc cùng họ với loài cần định danh, tất cả fasta đã được xếp giống cột và chung 1 file fasta. Đầu ra là cây di truyền hoàn chỉnh.
</div>

### 5.11. Chú giải genome (Bakta)

```bash
bakta \
  --input <path/to/input.fasta> \
  --output <path/to/output_dir> \
  --prefix SampleID \
  --threads 8
```

<div align="justify">
Bakta thực hiện chú giải: tìm CDS, gene RNA (rRNA, tRNA), CRISPR, chức năng protein, liên kết thư viện DBxref,… --prefix SampleID đặt tên tiền tố cho các file kết quả xuất ra. Ví dụ: Các file tạo ra sẽ có tên là SampleID.gff3, SampleID.gbk,... thay cho tên mặc định. Có thể đổi SampleID thành mã mẫu thực tế. Kết quả đầy đủ trong nhiều định dạng (GFF3, GenBank, FASTA protein/nt, bảng TSV báo cáo). Bakta thiết kế cho vi khuẩn và sử dụng phương pháp tìm kiếm không căn chỉnh nhanh, thường hoàn tất một genome ~5 Mb trong vài phút. Nên kiểm tra log của Bakta để biết các bản ghi (annotation) và phiên bản cơ sở dữ liệu đã dùng. 
- Thư mục `đầu ra` chứa các file:
  - `SampleID.gff` (chú giải GFF3 chuẩn GenBank),
  - `SampleID.gbk` (GenBank), `SampleID.embl` (EMBL),
  - `SampleID.ffn` (nucleotides of CDS), `SampleID.faa` (proteins),
  - `SampleID.tsv` (tóm tắt chức năng), `SampleID.json`, v.v.
</div>

### 5.12. Lắp ráp thay thế (Unicycler) [tùy chọn]

<div align="justify">
Unicycler – pipeline lắp ráp tập trung cho vi khuẩn (hỗ trợ tập short-reads và hybrid long-reads). Nếu kết quả lắp ráp SPAdes ở 5.4 không đạt yêu cầu (quá nhiều contigs, N50 thấp, hoặc nhiều lỗ hổng), có thể chạy Unicycler như một phương án thay thế với cùng dữ liệu short reads.
</div>

```bash
unicycler -1 <path/to/inputR1.fastq> \
          -2 <path/to/inputR2.fastq> \
          -o <path/to/output_dir> \
          -t 8
```

<div align="justify">
Unicycler lấy đầu vào tương tự SPAdes ở mục 5.4, đồng thời tự động sử dụng SPAdes ở chế độ short-read rồi xử lý thêm để tối ưu lắp ráp. Tham số -t số luồng CPU. (Nếu có long reads, có thể thêm tham số -l longreads.fastq để chạy hybrid, nhưng trong quy trình này giả định chỉ có short reads.). Thư mục đầu ra bao gồm file assembly.fasta cùng file log. Sau khi có kết quả từ Unicycler, kiểm tra lại chất lượng bằng QUAST.
</div>

### 5.13. Scaffold theo tham chiếu (RagTag) [tùy chọn]

<div align="justify">
RagTag sẽ căn cứ vào genome tham chiếu để sắp xếp, định hướng và nối các contigs. Kết quả là fasta và file AGP (với thông tin gap). Tùy chọn -t số luồng. Thư mục đầu ra chứa fasta, file ragtag.agp (mô tả vị trí các gap), và log. Đây là bản assembly cuối cùng (nếu được chấp nhận). Sau khi có kết quả, kiểm tra lại chất lượng bằng QUAST.
</div>

```bash
ragtag.py scaffold <path/to/input_ref.fasta> \
                   <path/to/input_dir> \
                   -o <path/to/output_dir> -t 8
```

<div align="justify">
- RagTag nhận đầu vào gồm file fasta của genome tham chiếu và thư mục chứa các contig đầu ra của SPAdes hoặc Unicycler. RagTag sẽ căn cứ vào genome tham chiếu để sắp xếp, định hướng và nối các contigs. Kết quả là fasta và file AGP (với thông tin gap). Tùy chọn -t số luồng. Thư mục đầu ra chứa fasta, file ragtag.agp (mô tả vị trí các gap), và log. Đây là bản assembly cuối cùng (nếu được chấp nhận). Sau khi có kết quả, kiểm tra lại chất lượng bằng QUAST.
  
- Chú ý: RagTag không khắc phục lỗi hoàn toàn, nhưng sắp xếp contigs tốt hơn nếu có reference phù hợp. Kiểm tra kỹ scaffold mới để tránh misassembly do reference không quá gần.
</div>

## Tiêu chí hoàn thành và điều kiện thực hiện

<div align="justify">
Quy trình kết thúc khi Bakta hoàn tất chú giải genome của mẫu đã chọn (contigs hoặc scaffolds cuối cùng). Lấy kết quả từ thư mục đầu ra của Bakta làm output cuối.
  
Kiểm tra assembly: Sau mỗi assembly (SPAdes, Unicycler, RagTag), luôn đánh giá với QUAST. Nếu các chỉ số (ví dụ số contig quá nhiều, N50 thấp, tổng kích thước bất thường, misassemblies) không chấp nhận được, chuyển sang giải pháp thay thế.

Nếu SPAdes đạt yêu cầu có thể bỏ qua Unicycler/RagTag và tiếp tục.

Nếu SPAdes kém: chạy Unicycler để so sánh (thường làm sớm vì Unicycler có thể cho assembly tốt hơn với short-reads).

Nếu vẫn phân mảnh & có reference phù hợp: chạy RagTag.

Tham chiếu (FastANI, RagTag): Nếu FastANI (Bước 12) cho giá trị thấp (<90-95%), có thể genome không cùng loài – khi đó cần tìm reference khác hoặc thôi không dùng RagTag. Chỉ dùng RagTag nếu chắc chắn reference gần.

16S rRNA: Nếu Barrnap không tìm thấy 16S (vì assembly thiếu vùng rRNA), có thể thử ở assembly khác (Unicycler) hoặc kiểm tra why không lắp ráp được (16S thường lặp). Trường hợp không tìm thấy 16S ở cả hai assembly có thể do phân mảnh cao.

Lưu ý: Trình tự và cách chạy có thể điều chỉnh tùy theo dữ liệu và mục tiêu. Ví dụ, có thể chạy Unicycler song song để so sánh, nhưng chỉ dùng kết quả tốt nhất. Luôn sao lưu dữ liệu quan trọng và kiểm tra kỹ báo cáo tại mỗi bước để đảm bảo quy trình diễn ra đúng.
</div>

---

## Summary Table (English)

| Step | Tool     | Purpose                                      | Inputs                     | Outputs                                | Mandatory/Optional      | Key parameters                |
|------|----------|----------------------------------------------|--------------------------- |----------------------------------------|-------------------------|-------------------------------|
| 1    | FastQC   | Check raw read quality                       | Raw FASTQ files            | FastQC HTML/ZIP reports                | Mandatory               | `-t threads`                  |
| 1    | MultiQC  | Aggregate FastQC results                     | Folder of FastQC results   | `multiqc_report.html`, `multiqc_data/` | Mandatory               | —                             |
| 2    | fastp    | Trim adapters, filter low-quality reads      | Raw R1.fastq, R2.fastq     | Cleaned FASTQ files + HTML/JSON report | Mandatory               | `--detect_adapter_for_pe`, quality and length filters |
| 3    | FastQC   | Check trimmed-read quality                   | Cleaned FASTQ files        | FastQC reports                         | Mandatory               | `-t threads`                  |
| 3    | MultiQC  | Aggregate post-filter FastQC results         | Folder of FastQC results   | `multiqc_report.html`                  | Mandatory               | —                             |
| 4    | SeqKit   | Compute read statistics (count, length)      | Cleaned FASTQ files        | Stats table on stdout                  | Mandatory               | —                             |
| 5    | SPAdes   | De novo genome assembly                      | Cleaned R1.fastq, R2.fastq | `contigs.fasta` (assembly)             | Mandatory               | `--isolate`, `-t threads`, `-m memory` |
| 5    | QUAST    | Evaluate assembly quality                    | `contigs.fasta` (+ ref)    | Assembly quality reports               | Mandatory               | `-t threads`, `-r ref.fasta` |
| 6    | Barrnap  | Find rRNA genes (5S, 16S, 23S)               | `contigs.fasta`            | GFF3 with rRNA coordinates             | Mandatory               | `--kingdom bac`     |
| 7    | grep + bedtools | Extract 16S rRNA sequences from GFF   | `rrna.gff`                 | `16S.fasta`                            | Optional (if 16S found) | `grep "16S_rRNA"`, `bedtools getfasta -s -name` |
| 8    | SeqKit   | Stats on 16S sequences                       | `16S.fasta`                | Stats table on stdout                  | Optional                | —                             |
| 9    | FastANI  | Compute genome-wide ANI vs reference         | Assembly FASTA, ref FASTA  | ANI results (tab file)                 | Conditional (if ref.)   | `-q assembly -r reference -o output` |
| 10   | MEGA12   | Phylogenetic tree construction               | Multiple sequence alignment (MSA) FASTA file of 16S rRNA genes      | Complete phylogenetic tree file             | Conditional (For species identification) | `Desktop GUI (Manual execution)` |
| 11   | Bakta    | Annotate final genome (genes, proteins, etc.)| Final assembly FASTA       | GFF3, GenBank, FASTA, TSV, JSON      | Mandatory (final step) | `--input assembly.fa --output dir --prefix ID` |
| 12   | Unicycler| Alternative assembly (short-read only)       | Cleaned R1.fastq, R2.fastq | `assembly.fasta` (Unicycler output) | Optional             | `-1 reads -2 reads -o out -t threads` |
| 12   | QUAST    | Evaluate Unicycler assembly                  | Unicycler `assembly.fasta` | Assembly reports                    | Optional             | Same as Step 8                |
| 13   | RagTag   | Reference-guided scaffolding of contigs      | Assembly FASTA, ref FASTA  | `ragtag.scaffold.fasta`             | Conditional (if needed) | `ragtag.py scaffold ref.fa contigs.fa` |
| 13   | QUAST + SeqKit | Evaluate scaffolded assembly             | `ragtag.scaffold.fasta`   | Reports; contig stats (SeqKit)      | Conditional           | Same as Step 8 + count `N`    |
This table summarizes each step: **Tool**, **Purpose**, **Inputs/Outputs**, **When to run** (mandatory or conditional), and **Key parameters**. (Steps like Unicycler and RagTag are optional alternatives run only if needed. FastANI requires a reference genome.) The final step is always running Bakta on the chosen assembly to complete annotation.
