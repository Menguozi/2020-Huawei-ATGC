# 2020-Huawei-ATGC
f2fs: support age threshold based garbage collection
There are several issues in current background GC algorithm:
- valid blocks is one of key factors during cost overhead calculation,
so if segment has less valid block, however even its age is young or
it locates hot segment, CB algorithm will still choose the segment as
victim, it's not appropriate.
- GCed data/node will go to existing logs, no matter in-there datas'
update frequency is the same or not, it may mix hot and cold data
again.
- GC alloctor mainly use LFS type segment, it will cost free segment
more quickly.

This patch introduces a new algorithm named age threshold based
garbage collection to solve above issues, there are three steps
mainly:

1. select a source victim:
- set an age threshold, and select candidates beased threshold:
e.g.
 0 means youngest, 100 means oldest, if we set age threshold to 80
 then select dirty segments which has age in range of [80, 100] as
 candiddates;
- set candidate_ratio threshold, and select candidates based the
ratio, so that we can shrink candidates to those oldest segments;
- select target segment with fewest valid blocks in order to
migrate blocks with minimum cost;

2. select a target victim:
- select candidates beased age threshold;
- set candidate_radius threshold, search candidates whose age is
around source victims, searching radius should less than the
radius threshold.
- select target segment with most valid blocks in order to avoid
migrating current target segment.

3. merge valid blocks from source victim into target victim with
SSR alloctor.

Test steps:
- create 160 dirty segments:
 * half of them have 128 valid blocks per segment
 * left of them have 384 valid blocks per segment
- run background GC

Benefit: GC count and block movement count both decrease obviously:

- Before:
  - Valid: 86
  - Dirty: 1
  - Prefree: 11
  - Free: 6001 (6001)

GC calls: 162 (BG: 220)
  - data segments : 160 (160)
  - node segments : 2 (2)
Try to move 41454 blocks (BG: 41454)
  - data blocks : 40960 (40960)
  - node blocks : 494 (494)

IPU: 0 blocks
SSR: 0 blocks in 0 segments
LFS: 41364 blocks in 81 segments

- After:

  - Valid: 87
  - Dirty: 0
  - Prefree: 4
  - Free: 6008 (6008)

GC calls: 75 (BG: 76)
  - data segments : 74 (74)
  - node segments : 1 (1)
Try to move 12813 blocks (BG: 12813)
  - data blocks : 12544 (12544)
  - node blocks : 269 (269)

IPU: 0 blocks
SSR: 12032 blocks in 77 segments
LFS: 855 blocks in 2 segments

Signed-off-by: Chao Yu <yuchao0@huawei.com>
[Jaegeuk Kim: fix a bug along with pinfile in-mem segment & clean up]
Signed-off-by: Jaegeuk Kim <jaegeuk@kernel.org>
