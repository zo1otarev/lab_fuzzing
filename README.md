1. Поднял Kali

2. Оффтоп
sudo -i
mkdir /Fuzz
cd /Fuzz

#3. Докер (Хотел запускать там, но потом передумал)
#sudo apt-get update
#sudo apt-get install ca-certificates curl
#sudo install -m 0755 -d /etc/apt/keyrings
#curl -fsSL https://download.docker.com/linux/debian/gpg |
#  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
#echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian bookworm stable" | \
#  sudo tee /etc/apt/sources.list.d/docker.list 
#sudo apt-get update
#sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

3. AFL++
apt-get install -y afl++
git clone https://github.com/AFLplusplus/AFLplusplus.git
cd AFLplusplus
make
cd ..

4. ClamAV source
git clone https://github.com/Cisco-Talos/clamav.git

5. Собирает ClamAV
#Зависимости
apt-get update && apt-get install -y \
  `# install tools` \
  gcc make pkg-config python3 python3-pip python3-pytest valgrind cmake \
  `# install clamav dependencies` \
  check libbz2-dev libcurl4-openssl-dev libjson-c-dev libmilter-dev \
  libncurses5-dev libpcre2-dev libssl-dev libxml2-dev zlib1g-dev
apt-get install -y cargo rustc

#Build
cd clamav-main
mkdir build && cd build
cmake .. \
	-DCMAKE_C_COMPILER=/Fuzz/AFLplusplus/afl-clang-fast \
	-DCMAKE_CXX_COMPILER=/Fuzz/AFLplusplus/afl-clang-fast++ \
        -D CMAKE_INSTALL_PREFIX=/usr \
        -D CMAKE_INSTALL_LIBDIR=lib \
        -D APP_CONFIG_DIRECTORY=/etc/clamav \
        -D DATABASE_DIRECTORY=/var/lib/clamav \
        -D ENABLE_JSON_SHARED=OFF
cmake --build .
ctest
cmake --build . --target install

6. Настраиваем и обновляем базу данных антивируса
sed '8 s/Example//' /etc/clamav/freshclam.conf.sample > /etc/clamav/freshclam.conf 
sed '8 s/Example//' /etc/clamav/clamd.conf.sample > /etc/clamav/clamd.conf 

groupadd clamav
useradd -g clamav -s /bin/false -c "Clam Antivirus" clamav

export AFL_IGNORE_PROBLEMS=1

chown clamav:clamav /var/lib/clamav 

#Добавил зеркала обновлений, но все равно не захотел обновляться
cat << EOF >> /etc/clamav/freshclam.conf                    
PrivateMirror https://clamav-mirror.ru/
PrivateMirror https://mirror.truenetwork.ru/clamav/ 
PrivateMirror http://mirror.truenetwork.ru/clamav/ 
ScriptedUpdates no
EOF

#Принудительно
wget https://clamav-mirror.ru/main.cvd -P /var/lib/clamav/
wget https://clamav-mirror.ru/daily.cvd -P /var/lib/clamav/
wget https://clamav-mirror.ru/bytecode.cvd -P /var/lib/clamav/

freshclam

7. Создаем файлы 
cd /Fuzz
mkdir in
mkdir out

wget https://secure.eicar.org/eicar.com -P /Fuzz/in/
wget https://secure.eicar.org/eicar.com.txt -P /Fuzz/in/
wget https://www.virusanalyst.com/eicar.zip -P /Fuzz/in/
echo test > /Fuzz/in/test.tx
echo 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' > /Fuzz/in/eicar.sh
echo 'Это тестовый файл' > /Fuzz/in/eicar.test
echo 'sdfasfkjsdbhkjsdbhkjsdbvsdhkbvklsdbvlsdbv' > /Fuzz/in/sfsdlkfhsdfjhsl.sh
wget "https://s3.eu-central-2.wasabisys.com/malshare-samples/cf8/93b/c67/cf893bc673af9f8778aa8d7d25b5538021ceb3e97fb8c94167fa6967b6706b5e?X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=J7P96RTICJ6VW743HODY%2F20241121%2Feu-central-2%2Fs3%2Faws4_request&X-Amz-Date=20241121T175917Z&X-Amz-SignedHeaders=host&X-Amz-Expires=300&X-Amz-Signature=1e697800e29f3bce11c4bd2fa714ea3035344c53e9148234e287ee3ea3604102" -O /Fuzz/in/virus 

8. Фаззинг
──(root㉿kali)-[/Fuzz]
└─# afl-fuzz -i /Fuzz/in -o /Fuzz/out -t 50000+ -- clamscan @@
[+] Enabled environment variable AFL_IGNORE_PROBLEMS with value 1
afl-fuzz++4.21c based on afl by Michal Zalewski and a large online community
[+] AFL++ is maintained by Marc "van Hauser" Heuse, Dominik Maier, Andrea Fioraldi and Heiko "hexcoder" Eißfeldt
[+] AFL++ is open source, get it at https://github.com/AFLplusplus/AFLplusplus
[+] NOTE: AFL++ >= v3 has changed defaults and behaviours - see README.md
[+] No -M/-S set, autoconfiguring for "-S default"
[!] WARNING: Target binary called without a prefixed path, make sure you are fuzzing the right binary: clamscan
[*] Getting to work...
[+] Using exploration-based constant power schedule (EXPLORE)
[+] Enabled testcache with 50 MB
[+] Generating fuzz data with a length of min=1 max=1048576
[*] Checking core_pattern...
[!] WARNING: Could not check CPU scaling governor
[+] You have 2 CPU cores and 4 runnable tasks (utilization: 200%).
[!] WARNING: System under apparent load, performance may be spotty.
[*] Setting up output directories...
[+] Output directory exists but deemed OK to reuse.
[*] Deleting old session data...
[+] Output dir cleanup successful.
[*] Checking CPU core loadout...
[+] Found a free CPU core, try binding to #0.
[*] Scanning '/Fuzz/in'...
[!] WARNING: Test case '/Fuzz/in/virus' is too big (3.91 MB, limit is 1.00 MB), partial reading
[+] Loaded a total of 8 seeds.
[*] Creating hard links for all input files...
[*] Validating target binary...
[*] Spinning up the fork server...
[+] All right - new fork server model v1 is up.
[*] Target map size: 47254
[*] No auto-generated dictionary tokens to reuse.
[*] Attempting dry run with 'id:000000,time:0,execs:0,orig:eicar.com'...
[!] WARNING: instability detected during calibration
    len = 68, map size = 3585, exec speed = 27923446 us, hash = a493b90c8da28f81
[!] WARNING: Instrumentation output varies across runs.                                                
[*] Attempting dry run with 'id:000001,time:0,execs:0,orig:eicar.com.txt'...
[!] WARNING: instability detected during calibration
    len = 68, map size = 3585, exec speed = 28378078 us, hash = d5e5e93e51dd35cb
[!] WARNING: No new instrumentation output, test case may be useless.                                  
[!] WARNING: Instrumentation output varies across runs.
[*] Attempting dry run with 'id:000002,time:0,execs:0,orig:eicar.sh'...
[!] WARNING: instability detected during calibration
    len = 69, map size = 3661, exec speed = 27960995 us, hash = 74d93f63df30ec30
[!] WARNING: Instrumentation output varies across runs.                                                
[*] Attempting dry run with 'id:000003,time:0,execs:0,orig:eicar.test'...
[!] WARNING: instability detected during calibration
    len = 33, map size = 3678, exec speed = 28210334 us, hash = 1c5e390d0f322082
[!] WARNING: Instrumentation output varies across runs.                                                
[*] Attempting dry run with 'id:000004,time:0,execs:0,orig:eicar.zip'...
[!] WARNING: instability detected during calibration
    len = 358, map size = 3771, exec speed = 27883736 us, hash = 23b9d2bbcdff678a
[!] WARNING: Instrumentation output varies across runs.                                                
[*] Attempting dry run with 'id:000005,time:0,execs:0,orig:sfsdlkfhsdfjhsl.sh'...
[!] WARNING: instability detected during calibration
    len = 42, map size = 3655, exec speed = 27923769 us, hash = 643415b1614f0735
[!] WARNING: Instrumentation output varies across runs.                                                
[*] Attempting dry run with 'id:000006,time:0,execs:0,orig:test.txt'...
[!] WARNING: instability detected during calibration
    len = 5, map size = 3219, exec speed = 27991009 us, hash = 4dfe66d7fc1c6b2e
[!] WARNING: Instrumentation output varies across runs.                                                
[*] Attempting dry run with 'id:000007,time:0,execs:0,orig:virus'...
[!] WARNING: instability detected during calibration
    len = 1048576, map size = 5862, exec speed = 34482177 us, hash = 0cab6938ca098a2f
[!] WARNING: Instrumentation output varies across runs.                                                
[+] All test cases processed.
[!] WARNING: The target binary is pretty slow! See docs/fuzzing_in_depth.md#i-improve-the-speed
[!] WARNING: Some test cases are huge (1.00 MB) - see docs/fuzzing_in_depth.md#i-improve-the-speed
[!] WARNING: Some test cases look useless. Consider using a smaller set.
[+] He	re are some useful stats:

    Test case count : 5 favored, 8 variable, 0 ignored, 8 total
       Bitmap range : 3219 to 5862 bits (average: 3877.00 bits)
        Exec timing : 27.9M to 34.5M us (average: 28.8M us)

[*] -t option specified. We'll use an exec timeout of 41378 ms.
[+] All set and ready to roll!

#Ничего не получилось, за 2 часа ни одного нового пути

#Переборщил, вспомнил, что нам нужно 2 входных корпуса

9. Фазим заново
mkdir /Fuzz/new_in 
mkdir /Fuzz/new_out
wget https://secure.eicar.org/eicar.com -P /Fuzz/new_in/
echo 'Это тестовый файл' > /Fuzz/new_in/test.txt
afl-fuzz -i /Fuzz/new_in -o /Fuzz/new_out -t 50000+ -- clamscan @@

К сожалению все еще очень долгое время выполнения, за 1,5 часа нашлось всего-лишь 15 корпусов. 
Поэтому продолжаем фазинг.
На данный момент - exec speed : 0.03/sec (zzzz...) , что очень мало

           american fuzzy lop ++4.21c {default} (clamscan) [explore]           
┌─ process timing ────────────────────────────────────┬─ overall results ────┐
│        run time : 0 days, 1 hrs, 39 min, 3 sec      │  cycles done : 0     │
│   last new find : 0 days, 0 hrs, 7 min, 58 sec      │ corpus count : 16    │
│last saved crash : none seen yet                     │saved crashes : 0     │
│ last saved hang : none seen yet                     │  saved hangs : 0     │
├─ cycle progress ─────────────────────┬─ map coverage┴──────────────────────┤
│  now processing : 1.0 (6.2%)         │    map density : 7.78% / 8.04%      │
│  runs timed out : 0 (0.00%)          │ count coverage : 1.19 bits/tuple    │
├─ stage progress ─────────────────────┼─ findings in depth ─────────────────┤
│  now trying : havoc                  │ favored items : 2 (12.50%)          │
│ stage execs : 15/5120 (0.29%)        │  new edges on : 10 (62.50%)         │
│ total execs : 221                    │ total crashes : 0 (0 saved)         │
│  exec speed : 0.03/sec (zzzz...)     │  total tmouts : 0 (0 saved)         │
├─ fuzzing strategy yields ────────────┴─────────────┬─ item geometry ───────┤
│   bit flips : 0/0, 0/0, 0/0                        │    levels : 2         │
│  byte flips : 0/0, 0/0, 0/0                        │   pending : 16        │
│ arithmetics : 0/0, 0/0, 0/0                        │  pend fav : 2         │
│  known ints : 0/0, 0/0, 0/0                        │ own finds : 13        │
│  dictionary : 0/0, 0/0, 0/0, 0/0                   │  imported : 0         │
│havoc/splice : 0/0, 0/0                             │ stability : 99.95%    │
│py/custom/rq : unused, unused, unused, unused       ├───────────────────────┘
│    trim/eff : n/a, n/a                             │          [cpu000:100%]
└─ strategy: explore ────────── state: in progress ──┘

В итоге фаззил 12 часов 

          american fuzzy lop ++4.21c {default} (clamscan) [explore]           
┌─ process timing ────────────────────────────────────┬─ overall results ────┐
│        run time : 0 days, 12 hrs, 42 min, 36 sec    │  cycles done : 0     │
│   last new find : 0 days, 0 hrs, 2 min, 41 sec      │ corpus count : 84    │
│last saved crash : none seen yet                     │saved crashes : 0     │
│ last saved hang : none seen yet                     │  saved hangs : 0     │
├─ cycle progress ─────────────────────┬─ map coverage┴──────────────────────┤
│  now processing : 1.1 (1.2%)         │    map density : 7.78% / 8.12%      │
│  runs timed out : 0 (0.00%)          │ count coverage : 1.26 bits/tuple    │
├─ stage progress ─────────────────────┼─ findings in depth ─────────────────┤
│  now trying : havoc                  │ favored items : 2 (2.38%)           │
│ stage execs : 596/5120 (11.64%)      │  new edges on : 28 (33.33%)         │
│ total execs : 1610                   │ total crashes : 0 (0 saved)         │
│  exec speed : 0.00/sec (zzzz...)     │  total tmouts : 0 (0 saved)         │
├─ fuzzing strategy yields ────────────┴─────────────┬─ item geometry ───────┤
│   bit flips : 0/0, 0/0, 0/0                        │    levels : 2         │
│  byte flips : 0/0, 0/0, 0/0                        │   pending : 84        │
│ arithmetics : 0/0, 0/0, 0/0                        │  pend fav : 2         │
│  known ints : 0/0, 0/0, 0/0                        │ own finds : 82        │
│  dictionary : 0/0, 0/0, 0/0, 0/0                   │  imported : 0         │
│havoc/splice : 0/0, 0/0                             │ stability : 99.95%    │
│py/custom/rq : unused, unused, unused, unused       ├───────────────────────┘
│    trim/eff : n/a, n/a                             │          [cpu000:100%]
└─ strategy: explore ────────── state: in progress ──┘^C

+++ Testing aborted by user +++

[!] Stopped during the first cycle, results may be incomplete.
    (For info on resuming, see docs/README.md)
[+] We're done here. Have a nice day!



10. Покрытие
apt install lcov

cd clamav-main
cd build
cmake .. \
    -DBUILD_EXAMPLES=ON \
    -DCMAKE_C_FLAGS="--coverage" \
    -DCMAKE_CXX_FLAGS="--coverage" \
    -D CMAKE_INSTALL_PREFIX=/usr \
    -D CMAKE_INSTALL_LIBDIR=lib \
    -D APP_CONFIG_DIRECTORY=/etc/clamav \
    -D DATABASE_DIRECTORY=/var/lib/clamav \
    -D ENABLE_JSON_SHARED=OFF

cmake --build .
ctest
cmake --build . --target install

for file in /Fuzz/new_out/default/queue/*; do clamscan $file ; done
find /Fuzz/clamav-main/ -name *.gcda
lcov -o cov.info -c -d /Fuzz/clamav-main
cat /Fuzz/clamav-main/cov.info
genhtml -o cov_data /Fuzz/clamav-main/cov.info

Итог:
Overall coverage rate:
  source files: 321
  lines.......: 11.5% (9706 of 84340 lines)
  functions...: 16.0% (472 of 2945 functions)
Message summary:
  no messages were reported
