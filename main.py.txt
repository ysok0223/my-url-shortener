from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import RedirectResponse
from pydantic import BaseModel
import hashlib

app = FastAPI()

# URL 저장소 (메모리 DB)
url_db = {}
reverse_db = {}

class ShortenRequest(BaseModel):
    url: str

@app.get("/")
def read_root():
    return {"message": "URL Shortener Service is running!"}

@app.post("/shorten")
def shorten_url(payload: ShortenRequest, request: Request):
    original_url = payload.url.strip()
    
    # 1. 자기 서비스 URL 단축 방지 (조건 2)
    base_url = str(request.base_url).rstrip("/")
    host_name = request.url.hostname
    
    if original_url.startswith(base_url) or (host_name and host_name in original_url):
        raise HTTPException(status_code=400, detail="Cannot shorten self-service URL")

    # 2. 동일 URL 중복 처리 (조건 1)
    if original_url in reverse_db:
        existing_code = reverse_db[original_url]
        return {
            "shortCode": existing_code,
            "shortUrl": f"{base_url}/{existing_code}"
        }

    # 3. 신규 단축 코드 생성
    short_code = hashlib.md5(original_url.encode()).hexdigest()[:6]
    
    url_db[short_code] = original_url
    reverse_db[original_url] = short_code

    return {
        "shortCode": short_code,
        "shortUrl": f"{base_url}/{short_code}"
    }

@app.get("/{short_code}")
def redirect_to_original(short_code: str):
    if short_code in url_db:
        return RedirectResponse(url=url_db[short_code])
    raise HTTPException(status_code=404, detail="URL not found")
