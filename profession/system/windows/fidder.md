# Fidder 
***
- [Auto-Build 2021-02-07 12:35](https://github.com/BtbN/FFmpeg-Builds/releases)


- 1.SSL open
- 2.Url filter:m3u3/auth_key
- 3.Process filter:dingtalk

```shell script
.\ffmpeg -i "https://dtliving-pre.alicdn.com/live_hp/259ae09c-8fbc-4ccb-9c71-6b35bf8b4928_merge.m3u8?app_type=win&auth_key=1615360924-0-0-e91b024fc860281496aaf64ed6023610&cid=e5b012ef6c10389b49d0c369f0759688&token=4f355a3038792537a90424b3feddc603CTG9CbgW1Q0gMoj1b4wOOMf6911yAq-nsJ2rqV-QIQmzrP1XD-RmCWYBzMHPBu_M7nYaFRKrzRFMQ7dK4ctJ6Qib3of5TtJXK9Df5FzORzs=&token2=c108c5a384656a6c1ee212fe244699bbVxu0gmtGdA1n_1LKeioVSrVZZxDtLio7emdn27OOUDVfMgUlJbAGvMxe70G-TVBeQ_h2GhjGbVFxOuyeRlNbTvyZWY96MmDpVJpGyYb2gEA&version=5.3.6-Release.3896" -c copy -bsf:a aac_adtstoasc D:\Video\Java\jvm1.mp4
```