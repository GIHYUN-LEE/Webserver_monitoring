# 📊 웹서버 모니터링

## 프로젝트 개요
### 👥 구성원
<table>
  <tr>
    <td align="center">
      <a href="https://github.com/dlacowns21">
        <img src="https://github.com/dlacowns21.png" width="100px;" alt="dlacowns21"/><br />
        <sub><b>임채준</b></sub>
      </a>
    </td>
    <td align="center">
      <a href="https://github.com/GIHYUN-LEE">
        <img src="https://github.com/GIHYUN-LEE.png" width="100px;" alt="GIHYUN-LEE"/><br />
        <sub><b>이기현</b></sub>
      </a>
    </td>
  </tr>
</table>

이 프로젝트는 **crontab**으로 의도된 트래픽을 주기적으로 발생시키고, **awk**로 `access.log`를 분석해 **상태코드 통계**를 산출합니다.

---

## 🖥️ 서버 스펙
- **메모리:** 4,096 MB  
- **CPU:** 2 vCPU  
- **디스크:** 30 GB  
- **웹서버:** NGINX

---

## 🛠️ 과정

### 1) Nginx 설치 및 실행 확인
```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx
sudo systemctl status nginx
```

### 2) Crontab으로 오류 로그 생성
```bash
# 200 정상코드
* * * * * curl -s http://localhost/ > /dev/null

# 400 에러코드
*/2 * * * * /usr/bin/curl --http1.1 -s -H "Host: invalid host" http://127.0.0.1/ -o /dev/null

# 403 에러코드
* * * * * /usr/bin/curl -s http://127.0.0.1/err403 -o /dev/null
*/3 * * * * curl -s http://localhost/restricted/ > /dev/null 2>&1

# 404 에러코드
* * * * * /usr/bin/curl -s http://127.0.0.1/does-not-exist -o /dev/null
*/2 * * * * curl -s http://localhost/nonexistent > /dev/null 2>&1
*/5 * * * * curl -s -A "GoogleBot" http://localhost/test > /dev/null 2>&1
*/2 * * * * curl -s http://localhost/404error > /dev/null 2>&1
*/3 * * * * curl -s http://localhost/admin > /dev/null 2>&1
*/7 * * * * curl -s http://localhost/login.jsp > /dev/null 2>&1

# 405 Method Not Allowed: 매 3분 (설정한 엔드포인트)
*/3 * * * * /usr/bin/curl -s -X POST http://127.0.0.1/onlyget -o /dev/null
*/6 * * * * curl -X DELETE http://localhost/ > /dev/null 2>&1

# 414 URI Too Long: 매 5분 (아주 긴 URL)
*/5 * * * * /usr/bin/curl -s "http://127.0.0.1/$(/usr/bin/head -c 12000 /dev/zero | /usr/bin/tr '\0' 'a')" -o /dev/null

# 502 에러코드(서버 에러 생성)
*/2 * * * * curl -s http://localhost/test.php > /dev/null 2>&1
*/4 * * * * curl -s http://localhost/secret.php > /dev/null 2>&1
```
### 3) 로그 분석
#### 1. 분석 스크립트 생성
```bash
cd ~
nano log_analyzer.sh
```
#### 2. 스크립트 내용 작성
```bash
#!/bin/bash

LOG_FILE="/var/log/nginx/access.log"
ERROR_LOG="/var/log/nginx/error.log"

clear
echo "======================================"
echo "    Nginx 웹서버 로그 분석 도구"
echo "======================================"
echo "1. 서버 상태 요약"
echo "2. 보안 위협 분석"
echo "3. 성능 분석"
echo "4. 에러 분석"
echo "5. 트래픽 패턴 분석"
echo "6. 실시간 모니터링"
echo "7. 시간대별 상세 분석"
echo "0. 종료"
echo "======================================"
read -p "선택하세요 (0-7): " choice

while true; do
    case $choice in
        1)
            echo "=== 서버 상태 요약 ==="
            echo "분석 시간: $(date)"
            echo "로그 파일: $LOG_FILE"
            
            sudo awk 'BEGIN {
                total=0; success=0; client_error=0; server_error=0;
            }
            {
                total++
                if ($9 >= 200 && $9 < 300) success++
                else if ($9 >= 400 && $9 < 500) client_error++
                else if ($9 >= 500) server_error++
            }
            END {
                printf "총 요청 수: %d\n", total
                printf "성공 요청: %d (%.1f%%)\n", success, (success/total)*100
                printf "클라이언트 에러: %d (%.1f%%)\n", client_error, (client_error/total)*100
                printf "서버 에러: %d (%.1f%%)\n", server_error, (server_error/total)*100
                printf "전체 에러율: %.1f%%\n", ((client_error+server_error)/total)*100
            }' $LOG_FILE
            ;;
            
        2)
            echo "=== 보안 위협 분석 ==="
            echo "의심스러운 활동 탐지:"
            
            # 404 에러가 많은 IP (스캐닝 의심)
            echo -e "\n[404 에러 다발 IP - 스캐닝 의심]"
            sudo awk '$9 == 404 {ip[$1]++} END {for(i in ip) if(ip[i] >= 3) print i, ip[i] "회"}' $LOG_FILE | sort -k2 -nr
            
            # 다양한 경로 시도 (해킹 시도 의심)
            echo -e "\n[다양한 경로 시도 IP]"
            sudo awk '{ip_path[$1][$7]++; ip_count[$1]++} END {for(ip in ip_count) if(ip_count[ip] >= 5) {paths=0; for(path in ip_path[ip]) paths++; if(paths >= 3) print ip, ip_count[ip] "회", paths "개 경로"}}' $LOG_FILE
            
            # SQL Injection 시도 의심
            echo -e "\n[SQL Injection 시도 의심]"
            sudo grep -i "union\|select\|drop\|insert" $LOG_FILE | awk '{print $1, $7}' | head -5
            ;;
            
        3)
            echo "=== 성능 분석 ==="
            
            # 가장 많이 요청된 페이지
            echo "[가장 많이 요청된 페이지 Top 10]"
            sudo awk '{url[$7]++} END {for(u in url) print url[u], u}' $LOG_FILE | sort -nr | head -10
            
            # 큰 응답 크기 요청
            echo -e "\n[대용량 응답 요청 (1KB 이상)]"
            sudo awk '$10 > 1000 {print $1, $7, $10 "bytes"}' $LOG_FILE | head -10
            
            # 느린 요청 가능성 (502, 504 에러)
            echo -e "\n[성능 관련 에러]"
            sudo awk '$9 == 502 || $9 == 504 {print $4, $1, $7, $9}' $LOG_FILE
            ;;
            
        4)
            echo "=== 에러 분석 ==="
            
            # 에러 유형별 통계
            echo "[에러 상태 코드별 통계]"
            sudo awk '$9 >= 400 {status[$9]++} END {for(s in status) print s":", status[s]}' $LOG_FILE | sort
            
            # 가장 많은 에러를 발생시키는 IP
            echo -e "\n[에러 발생 IP Top 5]"
            sudo awk '$9 >= 400 {ip[$1]++} END {for(i in ip) print ip[i], i}' $LOG_FILE | sort -nr | head -5
            
            # 가장 많은 에러가 발생하는 URL
            echo -e "\n[에러 발생 URL Top 5]"
            sudo awk '$9 >= 400 {url[$7]++} END {for(u in url) print url[u], u}' $LOG_FILE | sort -nr | head -5
            
            # 시스템 에러 로그
            echo -e "\n[최근 시스템 에러 (error.log)]"
            sudo tail -5 $ERROR_LOG 2>/dev/null || echo "에러 로그 없음"
            ;;
            
        5)
            echo "=== 트래픽 패턴 분석 ==="
            
            # 시간대별 요청 수
            echo "[시간대별 요청 분포]"
            sudo awk '{
                split($4, datetime, ":")
                hour = datetime[2]
                count[hour]++
            } 
            END {
                for(h=0; h<24; h++) {
                    printf "%02d시: ", h
                    requests = count[sprintf("%02d", h)]
                    if(requests == "") requests = 0
                    printf "%d회\n", requests
                }
            }' $LOG_FILE
            
            # 요일별 분석 (최근 데이터만)
            echo -e "\n[User Agent 분석]"
            sudo awk -F'"' '{ua[$6]++} END {for(u in ua) print ua[u], u}' $LOG_FILE | sort -nr | head -5
            ;;
            
        6)
            echo "=== 실시간 모니터링 시작 ==="
            echo "Ctrl+C로 중단하세요"
            echo "시간     IP주소        상태  URL"
            echo "----------------------------------------"
            sudo tail -f $LOG_FILE | awk '{
                split($4, time, ":")
                printf "%s:%s ", time[2], time[3]
                printf "%-15s %s   %s\n", $1, $9, $7
                fflush()
            }'
            ;;
            
        7)
            echo "=== 시간대별 상세 분석 ==="
            read -p "분석할 시간 입력 (예: 14 = 14시): " target_hour
            
            echo "[$target_hour 시 상세 분석]"
            sudo awk -v hour="$target_hour" '
            BEGIN {
                split("", hour_data)
                total=0; errors=0;
            }
            {
                split($4, datetime, ":")
                if(datetime[2] == sprintf("%02d", hour)) {
                    total++
                    ip[$1]++
                    status[$9]++
                    url[$7]++
                    if($9 >= 400) errors++
                }
            }
            END {
                if(total > 0) {
                    printf "총 요청: %d회, 에러: %d회 (%.1f%%)\n", total, errors, (errors/total)*100
                    print "\n[상위 IP]"
                    for(i in ip) print ip[i], i | "sort -nr | head -3"
                    print "\n[상위 URL]"  
                    for(u in url) print url[u], u | "sort -nr | head -3"
                    print "\n[상태 코드]"
                    for(s in status) print s":", status[s]
                } else {
                    print "해당 시간대 데이터 없음"
                }
            }' $LOG_FILE
            ;;
            
        0)
            echo "분석 도구를 종료합니다."
            exit 0
            ;;
            
        *)
            echo "잘못된 선택입니다."
            ;;
    esac

    echo -e "\n계속하려면 Enter를 누르세요..."
    read
    
    clear
    echo "======================================"
    echo "    Nginx 웹서버 로그 분석 도구"
    echo "======================================"
    echo "1. 서버 상태 요약"
    echo "2. 보안 위협 분석"
    echo "3. 성능 분석"
    echo "4. 에러 분석"
    echo "5. 트래픽 패턴 분석"
    echo "6. 실시간 모니터링"
    echo "7. 시간대별 상세 분석"
    echo "0. 종료"
    echo "======================================"
    read -p "선택하세요 (0-7): " choice
done
```
#### 3. 권한 설정
```bash
chmod +x log_analyzer.sh
./log_analyzer.sh
```

### 4) 분석 결과
#### 1. 서버 상태 요약
<img width="638" height="646" alt="4(서버 상태 요약 기능)" src="https://github.com/user-attachments/assets/8638dbff-1c7f-43af-bed9-5b7de143a5d4" /><br>
#### 2. 보안 위협 분석
<img width="560" height="707" alt="5(보안 위협 분석)" src="https://github.com/user-attachments/assets/c9cb321c-c8ed-424a-a8fb-8171318e07dc" /><br>
#### 3. 성능 분석
<img width="618" height="1178" alt="6(성능 분석)" src="https://github.com/user-attachments/assets/b4eec205-f2d4-4b2e-a4a4-eb3486b85faa" /><br>
#### 4. 에러 분석
<img width="2022" height="1127" alt="7(에러 분석)" src="https://github.com/user-attachments/assets/a7616d6a-77bb-4efe-b6cf-b7d876094800" /><br>
#### 5. 트래픽 패턴 분석
<img width="587" height="1097" alt="8(트래픽 패턴 분석)" src="https://github.com/user-attachments/assets/868eff23-6a94-4c5b-8125-13baedea9759" /><br>
#### 6. 실시간 모니터링
<img width="633" height="752" alt="9(실시간 모니터링)" src="https://github.com/user-attachments/assets/d94df7a4-ccba-41d0-a687-57a7bb75b571" /><br>
#### 7. 시간대별 상세 분석
<img width="556" height="931" alt="10(시간대별 상세 분석)" src="https://github.com/user-attachments/assets/5bae9051-8357-4ab2-b6d0-2c56492f23b6" />


