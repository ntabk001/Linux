### Crontab

* Command line

```
crontab -e: tạo hoặc chỉnh sửa file crontab 
crontab -l: hiển thị file crontab 
crontab -r: xóa file crontab
```

### Cài đặt crontab

```
yum install cronie -y
service crond start
chkconfig crond on
```

### Cấu trúc của crontab

```
*     *     *     *     *     command to be executed
-     -     -     -     -
|     |     |     |     |
|     |     |     |     +----- day of week (0 - 6) (Sunday=0)
|     |     |     +------- month (1 - 12)
|     |     +--------- day of month (1 - 31)
|     +----------- hour (0 - 23)
+------------- min (0 - 59)
``` 

Ex:

```
0,30 * * * * command  # Chạy script 30 phút 1 lần:
0,15,30,45 * * * * command # Chạy script 15 phút 1 lần
0 3 * * * command # Chạy script vào 3 giờ sáng mỗi ngày
* * * * * sh /etc/backup.sh # chạy mỗi phút một lần
0 0 * * * sh /etc/backup.sh # chạy 24h một lần
> /dev/null 2>&1 # Disable email
```

`[Minute] [hour] [Day_of_the_Month] [Month_of_the_Year] [Day_of_the_Week] [command]`

* Config other user

`crontab -u username -e`

`crontab -u username -l`

`0 2 * * * /bin/sh backup.sh # Run crontab at 2h AM`

`0 5,17 * * * /scripts/script.sh #Run crontab at 5h AM, 17h PM`

`* * * * *  /scripts/script.sh # Run crontab every minutes`

`0 17 * * sun  /scripts/script.sh # Run script at 17h PM Sunday`

`*/10 * * * * /scripts/monitor.sh # Run crontab every 10 minutes`

`* * * jan,may,aug *  /script/script.sh # Run crontab at month`

`0 17 * * sun,fri  /script/script.sh # Run crontab at days`

`0 4,17 * * sun,mon /scripts/script.sh`

`* * * * * /scripts/script.sh; /scripts/scrit2.sh`

- Để thực hiện script vào chủ nhật đầu tiên của tháng ta phải sử dụng điều kiện kèm theo:

`0 2 * * sun  [ $(date +%d) -le 07 ] && /script/script.sh`


```
@yearly /scripts/script.sh
@monthly /scripts/script.sh
@weekly /bin/script.sh
@daily /scripts/script.sh
```

### 1. Schedule a cron to execute at 2am daily.

- This will be useful for scheduling database backup on a daily basis.

`0 2 * * * /bin/sh backup.sh`

- Asterisk (*) is used for matching all the records.

### 2. Schedule a cron to execute twice a day.

- Below example command will execute at 5 AM and 5 PM daily. You can specify multiple time stamps by comma-separated.

`0 5,17 * * * /scripts/script.sh`

### 3. Schedule a cron to execute on every minutes.

- Generally, we don’t require any script to execute on every minute but in some cases, you may need to configure it.

`* * * * *  /scripts/script.sh`

### 4. Schedule a cron to execute on every Sunday at 5 PM.

- This type of cron is useful for doing weekly tasks, like log rotation, etc.

`0 17 * * sun  /scripts/script.sh`

### 5. Schedule a cron to execute on every 10 minutes.

- If you want to run your script on 10 minutes interval, you can configure like below. These types of crons are useful for monitoring.
```
*/10 * * * * /scripts/monitor.sh
*/10: means to run every 10 minutes. Same as if you want to execute on every 5 minutes use */5.
```

### 6. Schedule a cron to execute on selected months.

- Sometimes we required scheduling a task to be executed for selected months only. Below example script will run in January, May and August months.

`* * * jan,may,aug *  /script/script.sh`

### 7. Schedule a cron to execute on selected days.

- If you required scheduling a task to be executed for selected days only. The below example will run on each Sunday and Friday at 5 PM.

`0 17 * * sun,fri  /script/script.sh`

### 8. Schedule a cron to execute on first sunday of every month.

- To schedule a script to execute a script on the first Sunday only is not possible by time parameter, But we can use the condition in command fields to do it.

`0 2 * * sun  [ $(date +%d) -le 07 ] && /script/script.sh`

### 9. Schedule a cron to execute on every four hours.

- If you want to run a script on 4 hours interval. It can be configured like below.

`0 */4 * * * /scripts/script.sh`

### 10. Schedule a cron to execute twice on every Sunday and Monday.

- To schedule a task to execute twice on Sunday and Monday only. Use the following settings to do it.

`0 4,17 * * sun,mon /scripts/script.sh`

### 11. Schedule a cron to execute on every 30 Seconds.

- To schedule a task to execute every 30 seconds is not possible by time parameters, But it can be done by schedule same cron twice as below.
```
* * * * * /scripts/script.sh
* * * * *  sleep 30; /scripts/script.sh
```
### 12. Schedule a multiple tasks in single cron.

- To configure multiple tasks with single cron, Can be done by separating tasks by the semicolon ( ; ).

`* * * * * /scripts/script.sh; /scripts/scrit2.sh`

### 13. Schedule tasks to execute on yearly ( @yearly ).

- @yearly timestamp is similar to “0 0 1 1 *“. It will execute a task on the first minute of every year, It may useful to send new year greetings

`@yearly /scripts/script.sh`

### 14. Schedule tasks to execute on monthly ( @monthly ).

- @monthly timestamp is similar to “0 0 1 * *“. It will execute a task in the first minute of the month. It may useful to do monthly tasks like paying the bills and invoicing to customers.

`@monthly /scripts/script.sh`

### 15. Schedule tasks to execute on Weekly ( @weekly ).

- @weekly timestamp is similar to “0 0 * * mon“. It will execute a task in the first minute of the week. It may useful to do weekly tasks like the cleanup of the system etc.

`@weekly /bin/script.sh`

### 16. Schedule tasks to execute on daily ( @daily ).

- @daily timestamp is similar to “0 0 * * *“. It will execute a task in the first minute of every day, It may useful to do daily tasks.

`@daily /scripts/script.sh`

### 17. Schedule tasks to execute on hourly ( @hourly ).

- @hourly timestamp is similar to “0 * * * *“. It will execute a task in the first minute of every hour, It may useful to do hourly tasks.

`@hourly /scripts/script.sh`

### 18. Schedule tasks to execute on system reboot ( @reboot ).

- @reboot is useful for those tasks which you want to run on your system startup. It will be the same as system startup scripts. It is useful for starting tasks in the background automatically.

`@reboot /scripts/script.sh`

### 19. Redirect Cron Results to specified email account.

- By default, cron sends details to the current user where cron is scheduled. If you want to redirect it to your other account, can be done by setup MAIL variable like below

```
# crontab -l
MAIL=bob
0 2 * * * /script/backup.sh
```

### 20. Taking backup of all crons to plain text file.

- I recommend keeping a backup of all jobs entry in a file. This will help you to recover crons in case of accidental deletion.

Check current scheduled cron:

```
# crontab -l
MAIL=rahul
0 2 * * * /script/backup.sh
```

Backup cron to text file:

```
# crontab -l > cron-backup.txt
# cat cron-backup.txt
MAIL=rahul
0 2 * * * /script/backup.sh
```

- Removing current scheduled cron:

```
# crontab -r
# crontab -l
no crontab for root
```

- Restore crons from text file:

```
# crontab cron-backup.txt
# crontab -l
MAIL=rahul
0 2 * * * /script/backup.sh
``

Thanks for reading this article, I hope it will help you to understand Crontab in Linux. For scheduling one time tasks you can also use Linux at command.