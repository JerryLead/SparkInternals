# ภายใน Spark (Spark Internals)

Apache Spark รุ่น: 1.0.2,
เอกสาร รุ่น: 1.0.2.0

## ผู้เขียน
| Weibo Id | Name |
|:-----------|:-------------|
|[@JerryLead](http://weibo.com/jerrylead) | Lijie Xu |

## แปลและเรียบเรียง
| Twitter | Name |
|:-----------|:-------------|
|[@AorJoa](https://twitter.com/AorJoa) | Bhuridech Sudsee |


## เกริ่นนำ

เอกสารนี้เป็นการพูดคุยแลกเปลี่ยนเกี่ยวการการออกแบบและใช้งานซอฟต์แวร์ Apache Spark ซึ่งจะโฟกัสไปที่เรื่องของหลักการออกแบบ, กลไกการทำงาน, สถาปัตยกรรมของโค้ด และรวมไปถึงการปรับแต่งประสิทธิภาพ นอกจากประเด็นเหล่านี้ก็จะมีการเปรียบเทียบในบางแง่มุมกับ Hadoop MapReduce ในส่วนของการออกแบบและการนำไปใช้งาน อย่างหนึ่งที่ผู้เขียนต้องการให้ทราบคือเอกสารนึ้ไม่ได้ต้องการใช้โค้ดเป็นส่วนนำไปสู่การอธิบายจึงจะไม่มีการอธิบายส่วนต่างๆของโค้ด แต่จะเน้นให้เข้าใจระบบโดยรวมที่ Spark ทำงานในลักษณะของการทำงานเป็นระบบ (อธิบายส่วนโน้นส่วนนี้ว่าทำงานประสานงานกันยังไง) ลักษณะวิธีการส่งงานที่เรียกว่า Spark Job จนกระทั่งถึงการทำงานจนงานเสร็จสิ้น

มีรูปแบบวิธีการหลายอย่างที่จะอธิบายระบบของคอมพิวเตอร์ แต่ผู้เขียนเลือกที่จะใช้ **problem-driven** หรือวิธีการขับเคลื่อนด้วยปัญหา ขั้นตอนแรกของคือการนำเสนอปัญหาที่เกิดขึ้นจากนั้นก็วิเคราะห์ข้อมูลทีละขั้นตอน แล้วจึงจะใช้ตัวอย่างที่มีทั่วๆไปของ Spark เพื่อเล่าถึงโมดูลของระบบและความต้องการของระบบเพื่อที่จะใช้สร้างและประมวลผล และเพื่อให้เห็นภาพรวมๆของระบบก็จะมีการเลือกส่วนเพื่ออธิบายรายละเอียดของการออกแบบและนำไปใช้งานสำหรับบางโมดูลของระบบ ซึ่งผู้เขียนก็เชื่อว่าวิธีนี้จะดีกว่าการที่มาไล่กระบวนการของระบบทีละส่วนตั้งแต่ต้น

จุดมุ่งหมายของเอกสารชุดนี้คือพวกที่มีความรู้หรือ Geek ที่อยากเข้าใจการทำงานเชิงลึกของ Apache Spark และเฟรมเวิร์คของระบบประมวลผลแบบกระจาย (Distributed computing) ตัวอื่นๆ

ผู้เขียนพยายามที่จะอัพเดทเอกสารตามรุ่นของ Spark ที่เปลี่ยนอย่างรวดเร็ว เนื่องจากชุมชนนักพัฒนาที่แอคทิฟมากๆ ผู้เขียนเลือกที่จะใช้เลขรุ่นหลักของ Spark มาใช้กับเลขที่รุ่นของเอกสาร (ตัวอย่างใช้ Apache Spark 1.0.2 เลยใช้เลขรุ่นของเอกสารเป็น 1.0.2.0)

สำหรับข้อถกเถียงทางวิชาการ สามารถติดตามได้ที่เปเปอร์ดุษฏีนิพนธ์ของ Matei และเปเปอร์อื่นๆ หรือว่าจะติดตามผู้เขียนก็ไปได้ที่ [บล๊อคภาษาจีน](http://www.cnblogs.com/jerrylead/archive/2013/04/27/Spark.html)

ผู้เขียนไม่สามารถเขียนเอกสารได้เสร็จในขณะนี้ ครั้งล่าสุดที่ได้เขียนคือราว 3 ปีที่แล้วตอนที่กำลังเรียนคอร์ส Andrew Ng's ML อยู่ ซึ่งตอนนั้นมีแรงบัลดาลใจที่จะทำ ตอนที่เขียนเอกสารนี้ผู้เขียนใช้เวลาเขียนเอกสารขึ้นมา 20 กว่าวันใช้ช่วงซัมเมอร์ เวลาส่วนใหญ่ที่ใช้ไปใช้กับการดีบั๊ก, วาดแผนภาพ, และจัดวางไอเดียให้ถูกที่ถูกทาง. ผู้เขียนหวังเป็นอย่างยิ่งว่าเอกสารนี้จะช่วยผู้อ่านได้


## เนื้อหา
เราจะเริ่มกันที่การสร้าง Spark Job และคุยกันถึงเรื่องว่ามันทำงานยังไง จากนั้นจึงจะอธิบายระบบที่เกี่ยวข้องและฟีเจอร์ของระบบที่ทำให้งานเราสามารถประมวลผลออกมาได้

1. [Overview](https://github.com/JerryLead/SparkInternals/blob/master/markdown/1-Overview.md) ภาพรวมของ Apache Spark
2. [Job logical plan](https://github.com/JerryLead/SparkInternals/blob/master/markdown/2-JobLogicalPlan.md) แผนเชิงตรรกะ : Logical plan (data dependency graph)
3. [Job physical plan](https://github.com/JerryLead/SparkInternals/blob/master/markdown/3-JobPhysicalPlan.md) แผนเชิงกายภาย : Physical plan
4. [Shuffle details](https://github.com/JerryLead/SparkInternals/blob/master/markdown/4-shuffleDetails.md) กระบวนการสับเปลี่ยน (Shuffle)
5. [Architecture](https://github.com/JerryLead/SparkInternals/blob/master/markdown/5-Architecture.md) กระบวนการประสานงานของโมดูลในระบบขณะประมวลผล
6. [Cache and Checkpoint](https://github.com/JerryLead/SparkInternals/blob/master/markdown/6-CacheAndCheckpoint.md)  Cache และ Checkpoint
7. [Broadcast](https://github.com/JerryLead/SparkInternals/blob/master/markdown/7-Broadcast.md) ฟีเจอร์ Broadcast
8. Job Scheduling TODO
9. Fault-tolerance TODO

เอกสารนี้เขียนด้วยภาษา Markdown, สำหรับเวอร์ชัน PDF ภาษาจีนสามารถดาวน์โหลด [ที่นี่](https://github.com/JerryLead/SparkInternals/tree/master/pdf).

ถ้าคุณใช้ Max OS X, เราขอแนะนำ [MacDown](http://macdown.uranusjr.com/) แล้วใช้ธีมของ github จะทำให้อ่านได้สะดวก

## ตัวอย่าง
บางตัวอย่างที่ผู้เขียนสร้างข้นเพื่อทดสอบระบบขณะที่เขียนจะอยู่ที่ [SparkLearning/src/internals](https://github.com/JerryLead/SparkLearning/tree/master/src/internals).

## Acknowledgement

**Note : ส่วนของกิตติกรรมประกาศจะไม่แปลครับ**

I appreciate the help from the following in providing solutions and ideas for some detailed issues:

- [@Andrew-Xia](http://weibo.com/u/1410938285) Participated in the discussion of BlockManager's implemetation's impact on broadcast(rdd).

- [@CrazyJVM](http://weibo.com/476691290) Participated in the discussion of BlockManager's implementation.

- [@王联辉](http://weibo.com/u/1685831233) Participated in the discussion of BlockManager's implementation.

Thanks to the following for complementing the document:

| Weibo Id | Chapter | Content | Revision status |
|:-----------|:-------------|:-------------|:-------------|
| [@OopsOutOfMemory](http://weibo.com/oopsoom) | Overview | Relation between workers and executors and [Summary on Spark Executor Driver's Resouce Management](http://blog.csdn.net/oopsoom/article/details/38763985) (in Chinese) | There's not yet a conclusion on this subject since its implementation is still changing, a link to the blog is added |

Thanks to the following for finding errors:

| Weibo Id | Chapter | Error/Issue | Revision status |
|:-----------|:-------------|:-------------|:-------------|
| [@Joshuawangzj](http://weibo.com/u/1619689670) | Overview | When multiple applications are running, multiple Backend process will be created | Corrected, but need to be confirmed. No idea on how to control the number of Backend processes |
| [@\_cs\_cm](http://weibo.com/u/1551746393) | Overview | Latest groupByKey() has removed the mapValues() operation, there's no MapValuesRDD generated | Fixed groupByKey() related diagrams and text |
| [@染染生起](http://weibo.com/u/2859927402) | JobLogicalPlan | N:N relation in FullDepedency N:N is a NarrowDependency | Modified the description of NarrowDependency into 3 different cases with detaild explaination, clearer than the 2 cases explaination before |
| [@zzl0](https://github.com/zzl0) | Fisrt four chapters | Lots of typos，such as "groupByKey has generated the 3 following RDDs"，should be 2. Check [pull request](https://github.com/JerryLead/SparkInternals/pull/3/files)。 | All fixed |
| [@左手牵右手TEL](http://weibo.com/w397090770) | Cache and Broadcast chapter | Lots of typos | All fixed |
| [@cloud-fan](https://github.com/cloud-fan) | JobLogicalPlan | Some arrows in the Cogroup() diagram should be colored red | All fixed |
| [@CrazyJvm](http://weibo.com/476691290) | Shuffle details | Starting from Spark 1.1, the default value for spark.shuffle.file.buffer.kb is 32k, not 100k | All fixed |

Special thanks to [@明风Andy](http://weibo.com/mingfengandy) for his great support.

Special thanks to the rockers (including researchers, developers and users) who participate in the design, implementation and discussion of big data systems.
