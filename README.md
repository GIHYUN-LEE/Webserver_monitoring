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

# 405 에러코드
*/3 * * * * /usr/bin/curl -s -X POST http://127.0.0.1/onlyget -o /dev/null
*/6 * * * * curl -X DELETE http://localhost/ > /dev/null 2>&1

# 414 에러코드
*/5 * * * * /usr/bin/curl -s "http://127.0.0.1/$(/usr/bin/head -c 12000 /dev/zero | /usr/bin/tr '\0' 'a')" -o /dev/null

# 502 에러코드
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
##### 1. 서버 상태 요약 기능

###### 기본 통계 분석

```bash
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
}' /var/log/nginx/access.log
```

- `BEGIN { }`: 스크립트 시작 전 변수 초기화
- `$9`: Nginx 로그의 9번째 필드(HTTP 상태 코드)
- `END { }`: 모든 라인 처리 후 결과 출력
- `printf`: 소수점 포맷팅 출력

##### 2. 보안 위협 분석 기능

###### 404 에러 다발 IP 탐지 (포트 스캐닝 의심)

```bash
sudo awk '$9 == 404 {ip[$1]++} END {
    for(i in ip) 
        if(ip[i] >= 3) 
            print i, ip[i] "회"
}' /var/log/nginx/access.log | sort -k2 -nr
```

- `$9 == 404`: 404 상태 코드만 필터링
- `if(ip[i] >= 3)`: 임계값 설정으로 의심 활동 탐지

##### 3. 성능 분석 기능

###### 인기 페이지 분석

```bash
sudo awk '{url[$7]++} END {
    for(u in url) print url[u], u
}' /var/log/nginx/access.log | sort -nr | head -10
```

- `$7`: 요청 URL 필드
- `sort -nr`: 숫자 역순 정렬
- `head -10`: 상위 10개만 출력

###### 대용량 응답 탐지

```bash
sudo awk '$10 > 1000 {
    print $1, $7, $10 "bytes"
}' /var/log/nginx/access.log | head -10
```

- `$10`: 응답 크기 필드
- 조건부 출력: 특정 임계값 초과 데이터만 표시

##### 4. 에러 분석 기능

###### 에러 상태 코드별 통계

```bash
sudo awk '$9 >= 400 {status[$9]++} END {
    for(s in status) print s":", status[s]
}' /var/log/nginx/access.log | sort
```

###### 에러 발생 IP 분석

```bash
sudo awk '$9 >= 400 {ip[$1]++} END {
    for(i in ip) print ip[i], i
}' /var/log/nginx/access.log | sort -nr | head -5
```

- `$9 >= 400`: 클라이언트/서버 에러만 필터링

##### 5. 트래픽 패턴 분석 기능

###### 시간대별 요청 분포

```bash
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
}' /var/log/nginx/access.log
```

- `split($4, datetime, ":")`: 날짜/시간 필드 분할
- `sprintf("%02d", h)`: 시간 포맷팅 (01, 02, ...)
- 배열 초기화 검증: 빈 값 처리

###### User-Agent 분석

```bash
sudo awk -F'"' '{ua[$6]++} END {
    for(u in ua) print ua[u], u
}' /var/log/nginx/access.log | sort -nr | head -5
```

- `F'"'`: 따옴표를 필드 구분자로 설정
- `$6`: User-Agent 필드 (따옴표 기준)

##### 6. 실시간 모니터링 기능

```bash
sudo tail -f /var/log/nginx/access.log | awk '{
    split($4, time, ":")
    printf "%s:%s ", time[2], time[3]
    printf "%-15s %s   %s\n", $1, $9, $7
    fflush()
}'
```

- `tail -f`: 실시간 로그 추적
- `fflush()`: 즉시 출력 (버퍼링 방지)
- `%-15s`: 왼쪽 정렬 포맷팅

##### 7. 시간대별 상세 분석 기능

```bash
sudo awk -v hour="$target_hour" '
BEGIN {
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
        for(i in ip) print ip[i], i | "sort -nr | head -3"
    }
}' /var/log/nginx/access.log
```

- `v hour="$target_hour"`: 외부 변수 전달
- 조건부 집계: 특정 시간대만 필터링
- 파이프 활용: `| "sort -nr | head -3"`

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

---

### 5) 발전 방향
- 실제 운영 서비스의 로그를 분석합니다.
- 분석한 로그 데이터를 시각화합니다.


