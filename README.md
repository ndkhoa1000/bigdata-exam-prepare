# bigdata-exam-prepare

---

**Preconfigured Hadoop instance (zip)** — Google Drive:
[https://drive.google.com/file/d/1gdxQmwJujaHysIUgGj8pEObecWYbIY_i/view?usp=sharing](https://drive.google.com/file/d/1gdxQmwJujaHysIUgGj8pEObecWYbIY_i/view?usp=sharing)

**Install manual (Firefox or via `gdown`)**
(install `gdown` with `pip` if needed)

```bash
gdown https://drive.google.com/uc?id=1gdxQmwJujaHysIUgGj8pEObecWYbIY_i
```

---

Ghi chú cài Hadoop thủ công, file mẫu MapReduce (dùng Maven) và Makefile để đơn giản hoá việc test:
[https://github.com/ndkhoa1000/hadoop-cluster-installation](https://github.com/ndkhoa1000/hadoop-cluster-installation)

---

# 🐘 HADOOP MAPREDUCE CHEAT SHEET

**(Cheatsheet cho MapReduce — tổng hợp các mẫu thiết kế phổ biến + test mẫu)**

## 📚 Mục lục

1. [Filtering Pattern (Bộ lọc)](#filtering)
2. [Numerical Aggregation (Thống kê số liệu)](#aggregation)
3. [Inverted Index (Chỉ mục ngược)](#inverted)
4. [Distinct Pattern (Loại bỏ trùng lặp)](#distinct)
5. [Top-K Pattern (Tìm Top N)](#topk)
6. [Driver Configuration (Cấu hình Job)](#driver)

---

<a name="filtering"></a>

# 1. 🔍 FILTERING PATTERN (Bộ Lọc)

**Mục đích:** Tương tự mệnh đề `WHERE` trong SQL
**Đặc điểm:** Chỉ cần Mapper → `job.setNumReduceTasks(0)`

### ✅ Ví dụ 1 (Dễ): Lọc theo từ khóa (Log Analysis) — tìm dòng chứa `"ERROR"`

**Input**

```
INFO: Server started successfully.
ERROR: Database connection failed.
WARN: High memory usage.
ERROR: NullPointerException at line 42.
```

**Output mong đợi**

```
ERROR: Database connection failed.
ERROR: NullPointerException at line 42.
```

**Mapper**

```java
public void map(Object key, Text value, Context context) {
    String line = value.toString();
    if (line.contains("ERROR")) {
        context.write(value, NullWritable.get());
    }
}
```

---

### ✅ Ví dụ 2 (Trung bình): Lọc theo điều kiện số học — giao dịch > 1000$

**Input**

```
TX01,UserA,500.0
TX02,UserB,1500.0
TX03,UserC,200.0
TX04,UserD,5000.0
```

**Output mong đợi**

```
TX02,UserB,1500.0
TX04,UserD,5000.0
```

**Mapper**

```java
public void map(Object key, Text value, Context context) {
    String[] parts = value.toString().split(",");
    if (parts.length >= 3) {
        try {
            double amount = Double.parseDouble(parts[2]);
            if (amount > 1000.0) {
                context.write(value, NullWritable.get());
            }
        } catch (NumberFormatException e) {}
    }
}
```

---

### ✅ Ví dụ 3 (Trung bình): Lọc theo khoảng thời gian — chỉ dữ liệu năm 2024

**Input**

```
2023-12-31, Sales: 100
2024-01-01, Sales: 200
2024-05-20, Sales: 150
2022-10-10, Sales: 90
```

**Output mong đợi**

```
2024-01-01, Sales: 200
2024-05-20, Sales: 150
```

**Mapper**

```java
public void map(Object key, Text value, Context context) {
    String line = value.toString();
    if (line.startsWith("2024")) {
        context.write(value, NullWritable.get());
    }
}
```

---

<a name="aggregation"></a>

# 2. 📊 NUMERICAL AGGREGATION (Thống Kê Số Liệu)

**Mục đích:** GROUP BY, SUM, COUNT, AVG

### ✅ Ví dụ 1 (Dễ): Word Count

**Input**

```
Hello Hadoop
Hello World
```

**Output**

```
Hadoop  1
Hello   2
World   1
```

**Mapper**

```java
StringTokenizer itr = new StringTokenizer(value.toString());
while (itr.hasMoreTokens()) {
    word.set(itr.nextToken());
    context.write(word, new IntWritable(1));
}
```

**Reducer**

```java
int sum = 0;
for (IntWritable val : values) sum += val.get();
context.write(key, new IntWritable(sum));
```

---

### ✅ Ví dụ 2 (Trung bình): Tính Tổng Doanh Thu theo Store

**Input**

```
StoreA,100
StoreB,200
StoreA,300
```

**Output**

```
StoreA  400
StoreB  200
```

**Mapper**

```java
String[] parts = value.toString().split(",");
context.write(new Text(parts[0]), new IntWritable(Integer.parseInt(parts[1])));
```

**Reducer**

```java
int total = 0;
for (IntWritable val : values) total += val.get();
context.write(key, new IntWritable(total));
```

---

### ✅ Ví dụ 3 (Khó hơn): Tính điểm trung bình (Average)

**Input**

```
Math,8.0
Math,10.0
Physics,7.0
```

**Output**

```
Math    9.0
Physics 7.0
```

**Reducer**

```java
double sum = 0;
int count = 0;
for (DoubleWritable val : values) {
    sum += val.get();
    count++;
}
context.write(key, new DoubleWritable(sum / count));
```

---

<a name="inverted"></a>

# 3. 📖 INVERTED INDEX (Chỉ Mục Ngược)

**Mục đích:** Xây dựng chỉ mục cho tìm kiếm (term → file, vị trí)

### ✅ Ví dụ 1 (Dễ): Từ khóa → Tên file

**Input**

```
doc1.txt: "apple banana"
doc2.txt: "banana cherry"
```

**Output**

```
apple   doc1.txt
banana  doc1.txt, doc2.txt
cherry  doc2.txt
```

**Mapper**

```java
FileSplit fileSplit = (FileSplit) context.getInputSplit();
String fileName = fileSplit.getPath().getName();
context.write(new Text(word), new Text(fileName));
```

**Reducer**

```java
StringBuilder sb = new StringBuilder();
for (Text val : values) sb.append(val.toString()).append(", ");
context.write(key, new Text(sb.toString()));
```

---

### ✅ Ví dụ 2 (Trung bình): Từ khóa → File:LineNumber

**Input**

```
log.txt (line 10): "Error: Fail"
log.txt (line 50): "Error: Timeout"
```

**Output**

```
Error   log.txt:Line10, log.txt:Line50
```

**Mapper**

```java
long lineNum = ((LongWritable) key).get();
context.write(new Text(word), new Text(fileName + ":Line" + lineNum));
```

**Reducer:** nối string tương tự ví dụ trên

---

<a name="distinct"></a>

# 4. 🧹 DISTINCT PATTERN (Loại Bỏ Trùng Lặp)

**Mục đích:** SELECT DISTINCT / Dedup

### ✅ Ví dụ 1 (Dễ): Danh sách User ID duy nhất

**Input**

```
user1
user2
user1
user3
user2
```

**Output**

```
user1
user2
user3
```

**Mapper & Reducer**

```java
context.write(key, NullWritable.get());
```

---

### ✅ Ví dụ 2 (Trung bình): Loại bỏ dòng trùng lặp (Dedup)

**Input**

```
A, 10
B, 20
A, 10
```

**Output**

```
A, 10
B, 20
```

**Mapper**

```java
context.write(value, NullWritable.get());
```

---

<a name="topk"></a>

# 5. 🏆 TOP K PATTERN (Tìm Top N)

**Mục đích:** Tìm các phần tử nổi bật nhất (Top-K)

### ✅ Ví dụ 1 (Dễ): Top 2 từ dài nhất

**Input**

```
ant
hippopotamus
elephant
cat
```

**Output mong đợi**

```
hippopotamus
elephant
```

**Mapper (giữ cấu trúc TreeMap để track Top N)**

```java
topWords.put(word.length(), new Text(word));
if (topWords.size() > 2) topWords.remove(topWords.firstKey()); // Giữ Top 2
```

---

### ✅ Ví dụ 2 (Trung bình): Top sản phẩm bán chạy nhất

**Input**

```
Phone, 500
Laptop, 1000
Mouse, 20
```

**Output mong đợi**

```
Laptop (1000)
```

**Mapper**

```java
topProducts.put(sales, new Text(product));
if (topProducts.size() > 1) topProducts.remove(topProducts.firstKey());
```

---

<a name="driver"></a>

# ⚙️ DRIVER CONFIGURATION (Cấu hình Job)

* Set Output Value là số nguyên:

```java
job.setOutputValueClass(IntWritable.class);
```

* Set Output Value là số thực:

```java
job.setOutputValueClass(DoubleWritable.class);
```

* Map ra Int, nhưng Reduce ra Text (ví dụ MapOutput khác với Output):

```java
job.setMapOutputKeyClass(Text.class);
job.setMapOutputValueClass(IntWritable.class);
```

* Tối ưu hóa (Combiner):

```java
job.setCombinerClass(IntSumReducer.class);
```

* Job chỉ chạy Mapper (Lọc):

```java
job.setNumReduceTasks(0);
```

---

## 🔧 Thêm tài nguyên & công cụ (gợi ý)

* Google Drive preconfig zip (dùng `gdown` để tải): [https://drive.google.com/file/d/1gdxQmwJujaHysIUgGj8pEObecWYbIY_i/view?usp=sharing](https://drive.google.com/file/d/1gdxQmwJujaHysIUgGj8pEObecWYbIY_i/view?usp=sharing)
* Hướng dẫn cài + repo mẫu (Maven + Makefile): [https://github.com/ndkhoa1000/hadoop-cluster-installation](https://github.com/ndkhoa1000/hadoop-cluster-installation)
