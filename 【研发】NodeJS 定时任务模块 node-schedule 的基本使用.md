# NodeJS 定时任务模块 node-schedule 的基本使用

@ `node-schedule` 是 `Node.js` 的一个 定时任务（`crontab`）模块。我们可以使用定时任务来对服务器系统进行维护，
让其在固定的时间段执行某些必要的操作，还可以使用定时任务发送邮件、爬取数据等...

## 安装

[npm 地址](https://www.npmjs.com/package/node-schedule)

```
npm i -S node-schedule
```

## 基本使用

使用 `schedule.scheduleJob()` 即可创建一个定时任务，一个简单的上手示例：

```js
const schedule = require('node-schedule');

// 当前时间的秒值为 10 时执行任务，如：2018-7-8 13:25:10
let job = schedule.scheduleJob('10 * * * * *', () => {
    console.log(new Date());
});
```

`*` 的部分为时间数值规则，按以下规则表示：

```
* * * * * *
┬ ┬ ┬ ┬ ┬ ┬
│ │ │ │ │ |
│ │ │ │ │ └ day of week (0 - 7) (0 or 7 is Sun)
│ │ │ │ └───── month (1 - 12)
│ │ │ └────────── day of month (1 - 31)
│ │ └─────────────── hour (0 - 23)
│ └──────────────────── minute (0 - 59)
└───────────────────────── second (0 - 59, OPTIONAL)
```

6个占位符从左到右分别代表：秒、分、时、日、月、周几。

下面可以看看以下传入参数分别代表的意思：

```
每分钟的第30秒触发： '30 * * * * *'

每小时的1分30秒触发 ：'30 1 * * * *'

每天的凌晨1点1分30秒触发 ：'30 1 1 * * *'

每月的1日1点1分30秒触发 ：'30 1 1 1 * *'

2016年的1月1日1点1分30秒触发 ：'30 1 1 1 2016 *'

每周1的1点1分30秒触发 ：'30 1 1 * * 1'
```

## 进阶用法

除了基础的用法，我们还可以使用一些更为灵活的方法来实现定时任务。

**隔一段时间执行一次**:

```js
const schedule = require('node-schedule');

// 定义规则
let rule = new schedule.RecurrenceRule();

rule.second = [0, 10, 20, 30, 40, 50]; // 每隔 10 秒执行一次

// 启动任务
let job = schedule.scheduleJob(rule, () => {
    console.log(new Date());
});
rule 支持设置的值有 second、minute、hour、date、dayOfWeek、month、year 等。一些厂家的用法，如：

// 每秒执行
rule.second = [0,1,2,3......59];

// 每分钟 0 秒执行
rule.second = 0;

// 每小时 30 分执行
rule.minute = 30;
rule.second = 0;

// 每天 0 点执行
rule.hour =0;
rule.minute =0;
rule.second =0;

// 每月 1 号的 10 点执行
rule.date = 1;
rule.hour = 10;
rule.minute = 0;
rule.second = 0;

// 每周一、周三、周五的 0 点和 12 点执行
rule.dayOfWeek = [1,3,5];
rule.hour = [0,12];
rule.minute = 0;
rule.second = 0;
```

### 日志创建示例

本站点每天0点之前准时创建一个新的日志文件：

```js
import schedule from 'node-schedule';
import {join} from 'path';
import fs from 'fs';
import util from 'util';
import dayjs from 'dayjs';

const writeFile = util.promisify(fs.writeFile);

const rule = new schedule.RecurrenceRule();

// rule.date = [1];
// rule.dayOfWeek = [1,3,5];

rule.hour = [23];
rule.minute = [59];
rule.second = 50;

schedule.scheduleJob(rule, async () => {
    const logPath = join(__dirname, '..', 'logs', dayjs(Date.now() + 1000 * 20).format('YYYY-MM-DD') + '.json');
    try {
        await stat(logPath);
    } catch (error) {
        await writeFile(logPath, JSON.stringify([]));
    }
});
```

## 取消任务

可以使用 `job.cancel()` 终止一个运行中的任务。

```js
const job = schedule.scheduleJob();

job.cancel();
```

end!