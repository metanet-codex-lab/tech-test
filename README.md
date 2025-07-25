# MCP Toolbox Streamable HTTP Quickstart

이 가이드는 [genai-toolbox](https://github.com/googleapis/genai-toolbox) 저장소를 이용하여 MCP Toolbox 서버를 구축하고, MCP Inspector UI를 통해 Streamable HTTP(STTP) 방식으로 연결하는 기본적인 절차를 정리한 것입니다.

## 1. 저장소 클론

```bash
git clone https://github.com/googleapis/genai-toolbox
```

## 2. Toolbox 설치

릴리스 페이지에서 OS와 CPU 환경에 맞는 binary 를 내려받아 실행 권한을 부여합니다.

```bash
export VERSION=0.10.0
curl -O https://storage.googleapis.com/genai-toolbox/v$VERSION/linux/amd64/toolbox
chmod +x toolbox
```

또는 다음 명령으로 직접 빌드할 수도 있습니다.

```bash
go install github.com/googleapis/genai-toolbox@v0.10.0
```

## 3. `tools.yaml` 작성

아래 예시는 간단한 PostgreSQL 예제 구성입니다. DB 접속 정보는 환경에 맞게 수정하세요.

```yaml
sources:
  my-pg-source:
    kind: postgres
    host: 127.0.0.1
    port: 5432
    database: toolbox_db
    user: toolbox_user
    password: my-password

tools:
  search-hotels-by-name:
    kind: postgres-sql
    source: my-pg-source
    description: Search for hotels based on name.
    parameters:
      - name: name
        type: string
        description: The name of the hotel.
    statement: SELECT * FROM hotels WHERE name ILIKE '%' || $1 || '%';
  search-hotels-by-location:
    kind: postgres-sql
    source: my-pg-source
    description: Search for hotels based on location.
    parameters:
      - name: location
        type: string
        description: The location of the hotel.
    statement: SELECT * FROM hotels WHERE location ILIKE '%' || $1 || '%';
  book-hotel:
    kind: postgres-sql
    source: my-pg-source
    description: |-
      Book a hotel by its ID. If the hotel is successfully booked, returns a NULL, raises an error if not.
    parameters:
      - name: hotel_id
        type: string
        description: The ID of the hotel to book.
    statement: UPDATE hotels SET booked = B'1' WHERE id = $1;
  update-hotel:
    kind: postgres-sql
    source: my-pg-source
    description: |-
      Update a hotel's check-in and check-out dates by its ID. Returns a message indicating whether the hotel was successfully updated or not.
    parameters:
      - name: hotel_id
        type: string
        description: The ID of the hotel to update.
      - name: checkin_date
        type: string
        description: The new check-in date of the hotel.
      - name: checkout_date
        type: string
        description: The new check-out date of the hotel.
    statement: |
      UPDATE hotels SET checkin_date = CAST($2 as date), checkout_date = CAST($3 as date) WHERE id = $1;
  cancel-hotel:
    kind: postgres-sql
    source: my-pg-source
    description: Cancel a hotel by its ID.
    parameters:
      - name: hotel_id
        type: string
        description: The ID of the hotel to cancel.
    statement: UPDATE hotels SET booked = B'0' WHERE id = $1;

toolsets:
  my-toolset:
    - search-hotels-by-name
    - search-hotels-by-location
    - book-hotel
    - update-hotel
    - cancel-hotel
```

## 4. 서버 실행

```bash
./toolbox --tools-file "tools.yaml"
```

## 5. MCP Inspector 실행 및 STTP 연결

새 터미널에서 MCP Inspector를 실행합니다.

```bash
npx @modelcontextprotocol/inspector
```

실행 시 출력되는 `Session token`을 복사한 뒤 다음 링크를 브라우저에서 엽니다.

```
http://localhost:6274/?MCP_PROXY_AUTH_TOKEN=<YOUR_SESSION_TOKEN>
```

UI에서 아래 값을 설정합니다.

- **Transport Type:** `Streamable HTTP`
- **URL:** `http://127.0.0.1:5000/mcp`
- **Proxy Session Token:** 위에서 복사한 세션 토큰

`Connect` 버튼을 누르면 STTP 방식으로 Toolbox와 연결되며, `List Tools` 메뉴를 통해 설정된 도구를 확인하고 테스트할 수 있습니다.

