# Physical Plan

เราเคยสรุปสั้นๆ เกี่ยวกับกลไกของระบบคล้ายกับ DAG ใน Physical plan ซี่งรวมไปถึง Stage และ Task ด้วย ในบทนี้เราจะคุยกันถึงว่า **ทำอย่างไร Physical plan (Stage และ Task) จะถูกสร้างโดยให้ Logical plan ของแอพลิเคชันใน Spark**

## ความซับซ้อนของ Logical Plan

![ComplexJob](../PNGfigures/ComplexJob.png)
โค้ดของแอพพลิเคชันนี้จะแนบไว้ท้ายบท

**ทำอย่างไรถึงจะกำหนด Stage และ Task ได้อย่างเหมาะสม เช่น ความซับซ้อนของกราฟการขึ้นต่อกันของข้อมูล?**
ไอเดียอย่างง่ายๆที่เกี่ยวกับความเกี่ยวข้องระหว่าง RDD หนึ่งกับอีก RDD หนึ่งที่เกิดขึ้นก่อนหน้ามันจะอยู่ในรูปแบบของ Stage ซึ่งเป็นการอธิบายโดยใช้ทิศทางหัวลูกศร อย่างในแผนภาพด้านบนก็จะกลายมาเป็น Task. ในกรณีที่ RDD 2 ตัวรวมเข้ามาเป็นตัวเดียวกันนั้นเราสามารถสร้าง Stage โดยใช้ 3 RDD ซึ่งวิธีนี้ใช้ได้ก็จริงแต่ไม่ค่อยมีประสิทธิภาพ มันบอบบางและอาจจะก่อให้เกิดปัญหา **Intermediate data จำนวนมากซึ่งต้องการที่เก็บ** สำหรับ Physical task แล้วผลลัพธ์ของตัวมันจะถูกเก็บทั้งในโลคอลดิสก์ หรือในหน่วยความจำ หรือเก็บไว้ทั้งสองที่ สำหรับ Task ที่ถูกสร้างเพื่อแทนหัวลูกศรแต่ละตัวในกราฟการขึ้นต่อกันของข้อมูลระบบจะต้องเก็บข้อมูลของ RDD ทุกตัวไว้ ซึ่งทำให้สิ้นเปลืองทรัพยากรมาก

ถ้าเราตรวจสอบ Logical plan อย่างใกล้ชิดเราก็จะพบว่าในแต่ละ RDD นั้นพาร์ทิชันของมันจะไม่ขึ้นต่อกันกับตัวอื่น สิ่งที่ต้องการจะบอกก็คือใน RDD แต่ละตัวข้อมูลที่อยู่ในพาร์ทิชันจะไม่ยุ่งเกี่ยวกันเลย จากข้อสังเกตนี้จึงรวบรวมแนวความคิดเกี่ยวกับการรวมทั้งแผนภาพเข้ามาเป็น Stage เดียวและให้ Physical task เพื่อทำงานแค่ตัวเดียวสำหรับแต่ละพาร์ทิชันใน RDD สุดท้าย (`FlatMappedValuesRDD`) แผนภาพข้างล่างนี้จะทำให้เห็นแนวความคิดนี้ได้มากขึ้น

![ComplexTask](../PNGfigures/ComplexTask.png)

ลูกศรเส้นหนาทั้งหมดที่อยู่ในแผนภาพจะเป็นของ Task1 ซึ่งมันจะสร้างให้ผลลัพธ์ของพาร์ทิชันแรกของ RDD ตัวสุดท้ายของ Job นั้น โปรดทราบว่าเพื่อที่จะคำนวณ `CoGroupedRDD` เราจะต้องรู้ค่าของพาร์ทิชันทุกตัวของ RDD ที่เกิดก่อนหน้ามันเนื่องจากมันเป็นลักษณะของ `ShuffleDependency` ดังนั้นการคำนวณที่เกิดขึ้นใน Task1 เราถือโอกาสที่จะคำนวณ `CoGroupedRDD` ในพาร์ทิชันที่สองและสามสำหรับ Task2 และ Task3 ตามลำดับ และผลลัพธ์จาก Task2 และ Task3 แทนด้วยลูกศรบเส้นบางและลูกศรเส้นประในแผนภาพ

อย่างไรก็ดีแนวความคิดนี้มีปัญหาอยู่สองอย่างคือ:
  - Task แรกจะมีขนาดใหญ่มากเพราะเกิดจาก `ShuffleDependency` เราจำเป็นต้องรู้ค่าของทุกพาร์ทิชันของ RDD ที่เกิดก่อนหน้า
  - ต้องใช้ขั้นตอนวิธีที่ฉลาดในการกำหนดว่าพาร์ทิชันไหนที่จะถูกแคช
  
แต่มีจุดหนึ่งที่เป็นข้อดีที่น่าสนใจของไอเดียนี้ก็คือ **Pipeline ของข้อมูลซึ่งข้อมูลจะถูกประมวลผลจริงก็ต่อเมื่อมันมีความจำเป็นที่จะใช้จริงๆ** ยกตัวอย่างใน Task แรก เราจะตรวจสอบย้อนกลัยจาก RDD ตัวสุดท้าย (`FlatMappedValuesRDD`) เพื่อดูว่า RDD ตัวไหนและพาร์ทิชันตัวไหนที่จำเป็นต้องรู้ค่าบ้าง แล้วถ้าระหว่าง RDD เป็นความสัมพันธ์แบบ `NarrowDependency` ตัว Intermediate data ก็ไม่จำเป็นที่จะต้องถูกเก็บไว้

Pipeline มันจะเข้าใจชัดเจนขี้นถ้าเราพิจารณาในมุมมองระดับเรคอร์ด แผนภาพต่อไปนี้จะนำเสนอรูปแบบการประเมินค่าสำหรับ RDD ที่เป็น `NarrowDependency`

![Dependency](../PNGfigures/pipeline.png)

รูปแบบแรก (Pipeline pattern) เทียบเท่ากับการทำงาน:

```scala
for (record <- records) {
  f(g(record))
}
```

พิจารณาเรคอร์ดเป็น Stream เราจะเห็นว่าไม่มี Intermediate result ที่จำเป็นจะต้องเก็บไว้ มีแค่ครั้งเดียวหลังจากที่ `f(g(record))` ทำงานเสร็จแล้วผลลัพธ์ของการทำงานถึงจะถูกเก็บและเรคอร์ดสามารถถูก Gabage Collected เก็บกวาดให้ได้ แต่สำหรับบางรูปแบบ เช่นรูปแบบที่สามมันไม่เป็นเช่นนั้น:

```scala
for (record <- records) {
  val fResult = f(record)
  store(fResult)  // need to store the intermediate result here
}

for (record <- fResult) {
  g(record)
  ...
}
```

ชัดเจนว่าผลลัพธ์ของฟังก์ชัน `f` จะถูกเก็บไว้ที่ไหนสักที่ก่อน

ทีนี้กลับไปดูปัญหาที่เราเจอเกี่ยวกับ Stage และ Task ปัญหาหลักที่พบในไอเดียนี้ก็คือเราไม่สามารถทำ Pipeline แล้วทำให้ข้อมูลไหลต่อกันได้จริงๆ ถ้ามันเป็น `ShuffleDependency` ดังนั้นเราจึงต้องหาวิธีตัดการไหลข้อมูลที่แต่ละ `ShuffleDependency` ออกจากกัน ซึ่งจะทำให้ Chain หรือสายของ RDD มันหลุดออกจากกัน แล้วต่อกันด้วย `NarrowDependency` แทนเพราะเรารู้ว่า `NarrowDependency` สามารถทำ Pipeline ได้ พอคิดได้อย่างนี้เราก็เลยแยก Logical plan ออกเป็น Stage เหมือนในแผนภาพนี้

![ComplextStage](../PNGfigures/ComplexJobStage.png)

กลยุทธ์ที่ใช้ในการสร้าง Stage คือ **ตรวจสอบย้อนกลับโดยเริ่มจาก RDD ตัวสุดท้าย แล้วเพิ่ม `NarrowDependency` ใน Stage ปัจจุบันจากนั้นแยก `ShuffleDependency` ออกเป็น Stage ใหม่ ซึ่งในแต่ละ Stage จะมีจำนวน Task ที่กำหนดโดยจำนวนพาร์ทิชันของ RDD ตัวสุดท้าย ของ Stage **

จากแผนภาพข้างบนเว้นหนาคือ Task และเนื่องจาก Stage จะถูกกำหนกย้อนกลับจาก RDD ตัวสุดท้าย Stage ตัวสุดท้ายเลยเป็น Stage 0 แล้วถึงมี Stage 1 และ Stage 2 ซึ่งเป็นพ่อแม่ของ Stage 0 ตามมา **ถ้า Stage ไหนให้ผลลัพธ์สุดท้ายออกมา Task ของมันจะมีชนิดเป็น `ResultTask` ในกรณีอื่นจะเป็น `ShuffleMapTask`** ชื่อของ `ShuffleMapTask` ได้มาจากการที่ผลลัพธ์ของมันจำถูกสับเปลี่ยนหรือ Shuffle ก่อนที่จะถูกส่งต่อไปทำงานที่ Stage ต่อไป ซึ่งลักษณะนี้เหมือนกับที่เกิดขึ้นในตัว Mapper ของ Hadoop MapReduce ส่วน  `ResultTask` สามารถมองเป็น Reducer ใน Hadoop ก็ได้ (ถ้ามันรับข้อมูลที่ถูกสับเปลี่ยนจาก Stage ก่อนหน้ามัน) หรืออาจจะดูเหมือน Mapper (ถ้า Stage นั้นไม่มีพ่อแม่)

แต่ปัญหาอีกอย่างหนึ่งยังคงอยู่ : `NarrowDependency` Chain สามารถ Pipeline ได้ แต่ในตัวอย่างแอพพลิเคชันที่เรานำเสนอเราแสดงเฉพาะ `OneToOneDependency` และ `RangeDependency` แล้ว `NarrowDependency` แบบอื่นหล่ะ?

เดี๋ยวเรากลับไปดูการทำผลคูณคาร์ทีเซียนที่เคยคุยกันไปแล้วในบทที่แล้ว ซึ่ง `NarrowDependency` ข้างในมันค่อนข้างซับซ้อน:

![cartesian](../PNGfigures/Cartesian.png)

โดยมี Stage อย่างนี้:

![cartesian](../PNGfigures/cartesianPipeline.png)

ลูกศรเส้นหนาแสดงถึง `ResultTask` เนื่องจาก Stage จะให้ผลลัพธ์สุดท้ายออกมาโดยตรงในแผนภาพด้านบนเรามี 6 `ResultTask` แต่มันแตกต่างกับ `OneToOneDependency` เพราะ `ResultTask` ในแต่ละ Job ต้องการรู้ค่า 3 RDD และอ่าน 2 บล๊อคข้อมูลการทำงานทุกอย่างที่ว่ามานี้ทำใน Task ตัวเดียว **จะเห็นว่าเราไม่สนใจเลยว่าจริงๆแล้ว `NarrowDependency` จะเป็นแบบ 1:1 หรือ N:N, `NarrowDependency` Chain สามารถเป็น Pipeline ได้เสมอ โดยที่จำนวนของ Task จะเท่ากับจำนวนพาร์ทิชันของ RDD ตัวสุดท้าย **


## การประมวลผลของ Physical Plan

เรามี Stage และ Task ปัญหาต่อไปคือ **Task จะถูกประมวลผลสำหรับผลลัพธ์สุดท้ายอย่างไร?**

กลับไปดูกันที่ Physical plan ของแอพพลิเคชันตัวอย่างในบทนี้แล้วนึกถึง Hadoop MapReduce ซึ่ง Task จะถูกประมวลผลตามลำดับ ฟังก์ชัน `map()` จะสร้างเอาท์พุทของฟังก์ชัน Map ซึ่งเป็นการรับพาร์ทิชันมาแล้วเขียนลงไปในดิสก์เก็บข้อมูล จากนั้นกระบวนการ `shuffle-sort-aggregate` จะถูกนำไปใช้เพื่อสร้างอินพุทให้กับฟังกืชัน Reduce จากนั้นฟังก์ชัน `reduce()` จะประมวลผลเพื่อให้ผลลัพธ์สุดท้ายออกมา กระบวนการเหล่านี้แสดงได้ตามแผนภาพด้านล่าง:

![MapReduce](../PNGfigures/MapReduce.png)

กระบวนการประมวลผลนี้ไม่สามารถใช้กับ Physical plan ของ Spark ได้ตรงๆเพราะ Physical plan ของ Hadoop MapReduce นั้นมันเป็นกลไลง่ายๆและกำหนดไว้ตายตัวโดยที่ไม่มีการ Pipeline ด้วย

ไอเดียหลักของการทำ Pipeline ก็คือ **ข้อมูลจะถูกประมวลผลจริงๆ ก็ต่อเมื่อมันต้องการจะใช้จริงๆในกระบวนการไหลของข้อมูล** เราจะเริ่มจากผลลัพธ์สุดท้าย (ดังนั้นจึงทราบอย่างชัดเจนว่าการคำนวณไหนบ้างที่ต้องการจริงๆ) จากนั้นตรวจสอบย้อนกลับตาม Chain ของ RDD เพื่อหาว่า RDD และพาร์ทิชันไหนที่เราต้องการรู้ค่าเพื่อใช้ในการคำนวณจนได้ผลลัพธ์สุดท้ายออกมา กรณีที่เจอคือส่วนใหญ่เราจะตรวจสอบย้อนกลับไปจนถึงพาร์ทิชันบางตัวของ RDD ที่อยู่ฝั่งซ้ายมือสุดและพวกบางพาร์ทิชันที่อยู่ซ้ายมือสุดนั่นแหละที่เราต้องการที่จะรู้ค่าเป็นอันดับแรก

**สำหรับ Stage ที่ไม่มีพ่อแม่ ตัวมันจะเป็น RDD ที่อยู่ซ้ายสุดโดยตัวมันเองซึ่งเราสามารถรู้ค่าได้โดยตรงอยู่แล้ว (มันไม่มีการขึ้นต่อกัน) และแต่ละเรคอร์ดที่รู้ค่าแล้วสามารถ Stream ต่อไปยังกระบวนการคำนวณที่ตามมาภายหลังมัน (Pipelining)** Chain การคำนวณอนุมาณได้ว่ามาจากขั้นตอนสุดท้าย (เพราะไล่จาก RDD ตัวสุดท้ายมา RDD ฝั่งซ้ายมือสุด) แต่กลไกการประมวลผลจริง Stream ของเรคอร์ดนั้นดำเนินไปข้างหน้า (ทำจริงจากซ้ายไปขวา แต่ตอนเช็คไล่จากขวาไปซ้าย) เรคอร์ดหนึ่งๆจถูกทำทั้ง Chain การประมวลผลก่อนที่จะเริ่มประมวลผลเรคอร์ดตัวอื่นต่อไป

สำหรับ Stage ที่มีพ่อแม่นั้นเราต้องประมวลผลที่ Stage พ่อแม่ของมันก่อนแล้วจึงดึงข้อมูลผ่านทาง Shuffle จากนั้นเมื่อพ่อแม่ประมวลผลเสร็จหนึ่งครั้งแล้วมันจะกลายเป็นกรณีที่เหมือนกัน Stage ที่ไม่มีพ่อแม่ (จะได้ไม่ต้องคำนวณซ้ำๆ)

> ในโค้ดแต่ละ RDD จะมีเมธอต `getDependency()` ประกาศการขึ้นต่อกันของข้อมูลของมัน, `compute()` เป็นเมธอตที่ดูแลการรับเรคอร์ดมาจาก Upstream (จาก RDD พ่อแม่หรือแหล่งเก็บข้อมูล) และนำลอจิกไปใช้กับเรคอร์ดนั้นๆ เราจะเห็นบ่อยมากในโคดที่เกี่ยวกับ RDD (ผู้แปล: ตัวนี้เคยเจอในโค้ดของ Spark ในส่วนที่เกี่ยวกับ RDD) `firstParent[T].iterator(split, context).map(f)` ตัว `firstParent` บอกว่าเป็น RDD ตัวแรกที่ RDD มันขึ้นอยู่ด้วย, `iterator()` แสดงว่าเรคอร์ดจะถูกใช้แบบหนึ่งต่อหนึ่ง, และ `map(f)` เรคอร์ดแต่ละตัวจะถูกนำประมวลผลตามลอจิกการคำนวณที่ถูกกำหนดไว้. สุดท้ายแล้วเมธอต `compute()` ก็จะคืนค่าที่เป็น Iterator เพื่อใช้สำหรับการประมวลผลถัดไป

สรุปย่อๆดังนี้ : **การคำนวณทั่วทั้ง Chain จะถูกสร้างโดยการตรวจสอบย้อนกลับจาก RDD ตัวสุดท้าย แต่ละ `ShuffleDependency` จะเป็นตัวแยก Stage แต่ละส่วนออกจากกัน และในแต่ละ Stage นั้น RDD แต่ละตัวจะมีเมธอต `compute()` ที่จะเรียกการประมวลผลบน RDD พ่อแม่ `parentRDD.itererator()` แล้วรับ Stream เรคอร์ดจาก Upstream **

โปรดทราบว่าเมธอต `compute()` จะถูกจองไว้สำหรับการคำนวณลิจิกที่สร้างเอาท์พุทออกจากโดยใช้ RDD พ่อแม่ของมันเท่านั้น. RDD จริงๆของพ่อแม่ที่ RDD มันขึ้นต่อจะถูกประกาศในเมธอต `getDependency()` และพาร์ทิชันจริงๆที่มันขึ้นต่อจะถูกประกาศไว้ในเมธอต `dependency.getParents()`

ลองดูผลคูณคาร์ทีเซียน `CartesianRDD` ในตัวอย่างนี้

```scala
 // RDD x is the cartesian product of RDD a and RDD b
 // RDD x = (RDD a).cartesian(RDD b)
 // Defines how many partitions RDD x should have, what are the types for each partition
 override def getPartitions: Array[Partition] = {
    // create the cross product split
    val array = new Array[Partition](rdd1.partitions.size * rdd2.partitions.size)
    for (s1 <- rdd1.partitions; s2 <- rdd2.partitions) {
      val idx = s1.index * numPartitionsInRdd2 + s2.index
      array(idx) = new CartesianPartition(idx, rdd1, rdd2, s1.index, s2.index)
    }
    array
  }

  // Defines the computation logic for each partition of RDD x (the result RDD)
  override def compute(split: Partition, context: TaskContext) = {
    val currSplit = split.asInstanceOf[CartesianPartition]
    // s1 shows that a partition in RDD x depends on one partition in RDD a
    // s2 shows that a partition in RDD x depends on one partition in RDD b
    for (x <- rdd1.iterator(currSplit.s1, context);
         y <- rdd2.iterator(currSplit.s2, context)) yield (x, y)
  }

  // Defines which are the dependent partitions and RDDs for partition i in RDD x
  //
  // RDD x depends on RDD a and RDD b, both in `NarrowDependency`
  // For the first dependency, partition i in RDD x depends on the partition with
  //     index `i / numPartitionsInRDD2` in RDD a
  // For the second dependency, partition i in RDD x depends on the partition with
  //     index `i % numPartitionsInRDD2` in RDD b
  override def getDependencies: Seq[Dependency[_]] = List(
    new NarrowDependency(rdd1) {
      def getParents(id: Int): Seq[Int] = List(id / numPartitionsInRdd2)
    },
    new NarrowDependency(rdd2) {
      def getParents(id: Int): Seq[Int] = List(id % numPartitionsInRdd2)
    }
  )
```

## การสร้าง Job

จนถึงตอนนี้เรามีความรู้เรื่อง Logical plan และ Plysical plan แล้ว **ทำอย่างไรและเมื่อไหร่ที่ Job จะถูกสร้าง? และจริงๆแล้ว Job มันคืออะไรกันแน่**

ตารางด้านล่างแล้วถึงประเภทของ [action()](http://spark.apache.org/docs/latest/programming-guide.html#actions) และคอลัมภ์ถัดมาคือเมธอต `processPartition()` มันใช้กำหนดว่าการประมวลผลกับเรคอร์ดในแต่ละพาร์ทิชันจะทำอย่างไรเพื่อให้ได้ผลลัพธ์ คอลัมภ์ที่สามคือเมธอต `resultHandler()` จะเป็นตัวกำหนดว่าจะประมวลผลกับผลลัพธ์บางส่วนที่ได้มาจากแต่ละพาร์ทิชันอย่างไรเพื่อให้ได้ผลลัพธ์สุดท้าย


| Action | finalRDD(records) => result | compute(results) |
|:---------| :-------|:-------|
| reduce(func) | (record1, record2) => result, (result, record i) => result | (result1, result 2) => result, (result, result i) => result
| collect() |Array[records] => result | Array[result] |
| count() | count(records) => result | sum(result) |
| foreach(f) | f(records) => result | Array[result] |
| take(n) | record (i<=n) => result | Array[result] |
| first() | record 1 => result | Array[result] |
| takeSample() | selected records => result | Array[result] |
| takeOrdered(n, [ordering]) | TopN(records) => result | TopN(results) |
| saveAsHadoopFile(path) | records => write(records) | null |
| countByKey() | (K, V) => Map(K, count(K)) | (Map, Map) => Map(K, count(K)) |

แต่ละครั้งที่มีการเรียก `action()` ในโปรแกรมไดรว์เวอร์ Job จะถูกสร้างขึ้น ยกตัวอย่าง เช่น `foreach()` การกระทำนี้จะเรียกใช้ `sc.runJob(this, (iter: Iterator[T]) => iter.foreach(f)))` เพื่อส่ง Job ไปยัง `DAGScheduler` และถ้ามีการเรียก `action()` อื่นๆอีกในโปรแกรมไดรว์เวอร์ระบบก็จะสร้า Job ใหม่และส่งเข้าไปใหม่อีกครั้ง ดังนั้นเราจึงจะมี Job ได้หลายตัวตาม `action()` ที่เราเรียกใช้ในโปรแกรมไดรว์เวอร์ นี่คือเหตุผลที่ว่าทำไมไดรว์เวอร์ของ Spark ถูกเรียกว่าแอพพลิเคชันมากกว่าเรียกว่า Job

Stage สุดท้ายของ Job จะสร้างผลลัพธ์ของแต่ละ Job ออกมา ยกตัวอย่างเช่นสำหรับ `GroupByTest` ที่เราเคยคุยกันในบทแรกนั้นจะเห็นว่ามี Job อยู่ 2 Job และมีผลลัพธ์อยู่ 2 เซ็ตผลลัพธ์ เมื่อ Job ถูกส่งเข้าสู่ `DAGScheduler` และนำไปวิเคราะห์ตามแผนการในการแบ่ง Stage แล้วส่ง Stage อันดับแรกเข้าไปซึ่งเป็น **Stage ที่ไม่มีพ่อแม่** เพื่อประมวลผล ซึ่งในกระบวนการนี้จำนวน Task จะถูกกำหนดและ Stage จะถูกประมวลผลหลังจากที่ Stage พ่อแม่ของมันถูกประมวลผลเสร็จไปแล้ว


## รายละเอียดของการส่ง Job

มาสรุปการวิเคราะห์ย่อๆสำหรับโค้ดที่ใช้ในการสร้าง Job และการส่ง Job เข้าไปทำงาน เราจะกลับมาคุยเรื่องนี้ในบทเรื่องสถาปัตยกรรม

1. `rdd.action()` เรียกใช้งาน `DAGScheduler.runJob(rdd, processPartition, resultHandler)` เพื่อสร้าง Job
2. `runJob()` รับจำนวนพาร์ทิชันและประเภทของ RDD สุดท้ายโดยเรียกเมธอต `rdd.getPartitions()` จากนั้นจะจัดสรรให้ `Array[Result](partitions.size)` เอาไว้สำหรับเก็บผลลัพธ์ตามจำนวนของพาร์ทิชัน
3. สุดท้ายแล้ว `runJob(rdd, cleanedFunc, partitions, allowLocal, resultHandler)` ใน `DAGScheduler` จะถูกเรียกเพื่อส่ง Job, `cleanedFunc` เป็นลักษณะ Closure-cleaned ของฟังก์ชัน `processPartition`. ในกรณีนี้ฟังก์ชันสามารถที่จะ Serialized ได้และสามารถส่งไปยังโหนด Worker ตัวอื่นได้ด้วย 
4. `DAGScheduler` มีเมธอต `runJob()` ที่เรียกใช้ `submitJob(rdd, func, partitions, allowLocal, resultHandler)` เพื่อส่ง Job ไปทำการประมวลผล
5. `submitJob()` รับค่า `jobId`, หลังจากนั้นจะห่อหรือ Warp ด้วยฟังก์ชันอีกทีหนึ่งแล้วถึงจะส่งข้อความ `JobSubmitted` ไปหา `DAGSchedulerEventProcessActor`. เมื่อ `DAGSchedulerEventProcessActor` ได้รับข้อความแล้ว Actor ก็จะเรียกใช้ `dagScheduler.handleJobSubmitted()` เพื่อจัดการกับ Job ที่ถูกส่งเข้ามาแล้ว นี่เป็นตัวอย่างของ Event-driven programming แบบหนึ่ง
6. `handleJobSubmitted()` อันดับแรกเลยจะเรียก `finalStage = newStage()` เพื่อสร้าง Stage แล้วจากนั้นก็จะ `submitStage(finalStage)`. ถ้า `finalStage` มีพ่อแม่ ตัว Stage พ่อแม่จะต้องถูกประมวลผลก่อนเป็นอันดับแรกในกรณีนี้ `finalStage` นั้นจริงๆแล้วถูกส่งโดย `submitWaitingStages()`.

`newStage()` แบ่ง RDD Chain ใน Stage อย่างไร ?
- เมธอต `newStage()` จะเรียกใช้ `getParentStages()` ของ RDD ตัวสุดท้ายเมื่อมีการสร้าง Stage ขึ้นมาใหม่ (`new Stage(...)`)
- `getParentStages()` จะเริ่มไล่จาก RDD ตัวสุดท้ายจากนั้นตรวจสอบย้อนกลับโดยใช้ Logical plan และมันจะเพิ่ม RDD ลงใน Stage ปัจจุบันถ้าหาก Stage นั้นเป็น `NarrowDependency` จนกระทั่งมันเจอว่ามี `ShuffleDependency` ระหว่าง RDD มันจะให้ RDD ตัวมันเองอยู่ฝั่งทางขวา (RDD หลังจากกระบวนการ Shuffle) จากนั้นก็จบ Stage ปัจจุบัน ทำแบบนี้ไล่ไปเรื่อยๆโดยเริ่มจาก RDD ที่อยู่ทางซ้ายมือของ RDD ที่มีการ Shuffle เพื่อสร้าง Stage อื่นขึ้นมาใหม่ (ดูรูปการแบ่ง Stage อาจจะเข้าใจมากขึ้น)
- เมื่อ `ShuffleMapStage` ถูกสร้างขึ้น RDD ตัวสุดท้ายของมันก็จะถูกลงทะเบียนหรือ Register `MapOutputTrackerMaster.registerShuffle(shuffleDep.shuffleId, rdd.partitions.size)`. นี่เป็นสิ่งที่สำคัญเนื่องจากว่ากระบวนการ Shuffle จำเป็นต้องรู้ว่าเอาท์พุทจาก `MapOuputTrackerMaster`

ตอนนี้มาดูว่า `submitStage(stage)` ส่ง Stage และ Task ได้อย่างไร:

1. `getMissingParentStages(stage)` จะถูกเรียกเพื่อกำหนด `missingParentStages` ของ Stage ปัจจุบัน ถ้า Stage พ่อแม่ทุกตัวของมันถูกประมวลผลไปแล้วตัว `missingParentStages` จะมีค่าว่างเปล่า
2. ถ้า `missingParentStages` ไม่ว่างเปล่าจะทำการวนส่ง Stage ที่หายไปเหล่านั้นซ้ำและ Stage ปะจุบันจะถูกแทรกเข้าไปใน `waitingStages` และเมื่อ Stage พ่อแม่ทำงานเรียบร้อยแล้ว Stage ที่อยู่ใน `waitingStages` จะถูกทำงาน
3. ถ้า `missingParentStages` ว่างเปล่าและเรารู้ว่า Stage สามารถถูกประมวลผลในขณะนี้ แล้ว `submitMissingTasks(stage, jobId)` จะถูกเรียกให้สร้างและส่ง Task จริงๆ และถ้า Stage เป็น `ShuffleMapStage` แล้วเราจะจัดสรร `ShuffleMapTask` จำนวนมากเท่ากับจำนวนของพาร์ทิชันใน RDD ตัวสุดท้าย ในกรณีที่เป็น `ResultStage`, `ResultTask` จะถูกจัดสรรแทน. Task ใน Stage จะฟอร์ม `TaskSet` ขึ้นมา จากนั้นขั้นตอนสุดท้าย `taskScheduler.submitTasks(taskSet)` จำถูกเรียกและส่งเซ็ตของ Task ทั้งหมดไป
4. ชนิดของ `taskScheduler` คือ `TaskSchedulerImpl`. ใน `submitTasks()` แต่ละ `taskSet` จะได้รับการ Wrap ในตัวแปร `manager` ซึ่งเป็นตัวแปรของชนิด `TaskSetManager` แล้วจึงส่งมันไปทำ `schedulableBuilder.addTaskSetManager(manager)`. `schedulableBuilder` สามารถเป็น `FIFOSchedulableBuilder` หรือ `FairSchedulableBuilder`, ขึ้นอยู่กับว่าการตั้งค่ากำหนดไว้ว่าอย่างไร จากนั้นขั้นตอนสุดท้ายของ `submitTasks()` คือแจ้ง `backend.reviveOffers()` เพื่อให้ Task ทำงาน. ชนิดของ Backend คือ `SchedulerBackend`. ถ้าแอพพลิเคชันทำงานอยู่บนคลัสเตอร์มันจะเป็น Backend แบบ `SparkDeploySchedulerBackend` แทน
5. `SparkDeploySchedulerBackend` เป็น Sub-Class ของ `CoarseGrainedSchedulerBackend`, `backend.reviveOffers()` จริงๆจะส่งข้อความ `ReviveOffers` ไปหา `DriverActor`. `SparkDeploySchedulerBackend` จะเปิด `DriverActor` ในขั้นตอนเริ่มทำงาน. เมื่อ `DriverActor` ได้รับข้อความ `ReviveOffers` แล้วมันจะเรียก `launchTasks(scheduler.resourceOffers(Seq(new WorkerOffer(executorId, executorHost(executorId), freeCores(executorId)))))` เพื่อเปิดให้ Task ทำงาน จากนั้น `scheduler.resourceOffers()` ได้รับ `TaskSetManager` ที่เรียงลำดับแล้วจากตัวจัดการงาน FIFO หรือ Fair และจะรวบรวมข้อมูลอื่นๆที่เกี่ยวกับ Task จาก `TaskSchedulerImpl.resourceOffer()`. ซึ่งข้อมูลเหล่านั้นจัดเก็บอยู่ใน `TaskDescription` ในขั้นตอนนี้ตำแหน่งที่ตั้งของข้อมูลหรือ Data locality ก็จะถูกพิจารณาด้วย
6. `launchTasks()` อยู่ใน `DriverActor` จะ Serialize แต่ละ Task ถ้าขนาดของ Serialize ไม่เกินลิมิตของ `akkaFrameSize` จานั้น Task จะถูกส่งครั้งสุดท้ายไปยัง  Executor เพื่อประมวลผล: `executorActor(task.executorId) ! LaunchTask(new SerializableBuffer(serializedTask))`.

## การพูดคุย
จนกระทั่งถึงตอนนี้เราคุยกันถึง:
- โปรแกรมไดร์เวอร์ Trigger Job ได้อย่างไร?
- จะสร้าง Physical plan จาก Logical plan ได้อย่างไร?
- อะไรคือการ Pipelining ใน Spark และจำนำมันไปใช้ได้อย่างไร?
- โค้ดจริงๆที่สร้าง Job และส่ง Job เข้าสู่ระบบ

อย่างไรก็ตามก็ยังมีหัวข้อที่ไม่ได้ลงรายละเอียดคือ:
- กระบวนการ Shuffle
- การประมวลผลของ Task และตำแหน่งที่มันถูกประมวลผล

ในหัวข้อถัดไปเราจะถกเถียงกันถึงการะบวนการ Shuffle ใน Spark

ในความเห็นของผู้แต่งแล้วการแปลงจาก Logical plan ไปยัง Physical plan นั้นเป็นผลงานชิ้นเอกอย่างแท้จริง สิ่งที่เป็นนามธรร เช่น การขึ้นต่อกัน, Stage และ Task ทั้งหมดนี้ถูกกำหนดไว้อย่างดีสำหรับลอจิคของขั้นตอนวิธีก็ชัดเจนมาก

## โค้ดจากตัวอย่างของ Job

```scala
package internals

import org.apache.spark.SparkContext
import org.apache.spark.SparkContext._
import org.apache.spark.HashPartitioner


object complexJob {
  def main(args: Array[String]) {

    val sc = new SparkContext("local", "ComplexJob test")

    val data1 = Array[(Int, Char)](
      (1, 'a'), (2, 'b'),
      (3, 'c'), (4, 'd'),
      (5, 'e'), (3, 'f'),
      (2, 'g'), (1, 'h'))
    val rangePairs1 = sc.parallelize(data1, 3)

    val hashPairs1 = rangePairs1.partitionBy(new HashPartitioner(3))


    val data2 = Array[(Int, String)]((1, "A"), (2, "B"),
      (3, "C"), (4, "D"))

    val pairs2 = sc.parallelize(data2, 2)
    val rangePairs2 = pairs2.map(x => (x._1, x._2.charAt(0)))


    val data3 = Array[(Int, Char)]((1, 'X'), (2, 'Y'))
    val rangePairs3 = sc.parallelize(data3, 2)


    val rangePairs = rangePairs2.union(rangePairs3)


    val result = hashPairs1.join(rangePairs)

    result.foreachWith(i => i)((x, i) => println("[result " + i + "] " + x))

    println(result.toDebugString)
  }
}
```
