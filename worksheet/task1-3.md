# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** สุทธนพ ชะนะ

**Student ID / รหัสนิสิต:** 6632239821

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 497 | 600 | 93.5% | 100% |
| t0-small | 11 | 23 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

>ใน large model มี accuracy เพิ่มขึ้น แต่ทั้งนี้ต้องแลกมากับ latency และ cost (แสดง 100% ในช่อง large calls)

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 28.4 | 225 | 336 | 20 | 0 | 92.8% |
| t1-after | 9.2 | 405 | 1009 | 0 | 25 | 93.7% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

>เนื่องจากค่าที่ได้วัดจาก request ที่ตอบกลับมาเท่านั้นทำให้ใน t1-before ที่กลายเป็น error ไม่ถูกนำมาพิจารณา สามารถดูได้จาก column errors และ run time ที่ค่อนนข้างมาก

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

>เนื่องจากการค้างที่ calls ไปยัง large model นั้นจะเป็น request ที่ค้างสะสมจนเต็ม pool และจะไม่คืน resources จนกว่าจะ timeout ทำให้ request ปกติไม่สามารถใช้งานได้ เมื่อเทียบกับการที่รีวิว 10% ช้าลงเฉยๆ ที่ถึงแม้ request จะช้า แต่ก็ทำงานจนจบ ทำให้ request อื่นๆ ยังสามารถใช้งานได้ปกติ

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

>อาจจะพิจารณาการเก็บ review ทไว้ใน Queue เพื่อที่เมื่อมีบางส่วนใช้เวลานานหรือขัดข้อง review จะยังคงอยู่ใน Queue และสามารถนำมา process ต่อได้เมื่อระบบกลับมาใช้งานได้ปกติ นั้นคือถ้า status = pending สามารถส่งเข้า broker แล้วกำหนดให้ worker มา label ทีหลังได้ เราสามารถใช้ Message Broker เป็น topology มาปรับใช้

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*
> timeout ที่เวลา 0.1 วินาทีไวเกินไป โดยทั่วไปแล้วหากพิจารณาจากผลลัพธ์ที่ได้ จะเห็นว่า request ส่วนมาก ใช้เวลาเกิน 0.1 วินาทีทั้งสิ้น ทำให้เกิด timeout ก่อนที่ request จะถูกประมวลผลเสร็จ และในกรณีที่ตั้ง timeout นานเกินไป (10 วินาที) ทำให้ request หนึ่งๆ ถือครอง resources นานเกินจำเป็นส่งผลให้ request อื่นๆไม่สามารถ process ได้ เกิด latency ที่สูงเกินจำเป็น

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 5 | 539 | 91.5% | 38% |
| t2-0.7 | 0.7 | 366 | 574 | 93.0% | 53% |
| t2-0.8 | 0.8 | 448 | 569 | 93.5% | 64% |
| t2-0.9 | 0.9 | 472 | 584 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

>threshold = 0.7 เนื่องจากเมื่อเทียบกับ baseline แล้วให้ค่า accuracy ที่ค่อนข้างใกล้เคียงกัน และเมื่อพิจารณา cost จากการเรียก large calls แล้วจะเห็นว่าที่ threshold = 0.8-0.9 ให้ accuracy ใกล้เคียงกันแต่ cost ในการเรียกต่างกันมาก ความต่างของ accuracy 0.5% ที่ยอมเสียไป จึงเป็น trade-off ที่ยอมรับได้ ถ้าเทียบกับ cost และ latency ที่ save ได้

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

>เนื่องจากโดยทั้วไปแล้ว request สามารถจบที่ small model ได้เลย โดยไม่ต้องเปลี่ยนไปใช้ large model ที่มี latency มากกว่า และกลุ่ม p95 คือตัวที่ small model ไม่มั่นใจและไปเรียกใช้ large model ทำให้มี latency สูงกว่า

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

>เมื่อคนรีวิวใช้คำแบบประชดประชัน การ classify ของ small model อาจเข้าใจผิดว่าเป็นการรีวิวในแง่ดีได้ ทำให้ผู้ใช้อาจเข้าใจผิดว่าร้านนี้ดี ทั้งที่จริงๆรีวิวเป็นด้านลบ

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 5 | 543 | 93.0% | 38% | 35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

>รีวิวที่เป็นคำที่รู้จักกันดี หรือ ถูกรีวิวบ่อยๆ จะได้ประโยชน์มาก ส่วนรีวิวที่ที่เป็นประโยคยาวๆ หรือ คำเฉพาะตัวที่ไม่ค่อยมีคใช้จะแทยไม่ได้ประโยชน์จาก cache เลย

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

>อาจจะปรับปรุง cache ใหม่ ไม่ให้ขึ้นกับ model เดิม โดยอาจจะใช้การทำ versionning หรือกำหนด TTL ให้เหมาะสม

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

>การอ่านค่า p50 อาจจะหลอกเราได้ ใน t1-before p50 ต่ำกว่า baseline ทั้งที่ 10% ล้มเหลว ต้องดู errors และ p95 ประกอบเสมอ
