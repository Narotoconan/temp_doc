### 异常处理 __init__

```python
from fastapi import HTTPException


class FailedError(HTTPException):
    def __init__(self, message: str = "Failed"):
        self.status_code = 200
        self.detail = message
```



### register

```python
from fastapi import Request
from fastapi.exceptions import RequestValidationError, StarletteHTTPException


def register(app):
    # HTTP请求错误处理
    @app.exception_handler(StarletteHTTPException)
    async def http_exception_handler(request: Request, exc):
        return await request_http_exception_handler(request, exc)
```

### handler

```python
from fastapi import Request
from app.core.result import Result


async def request_http_exception_handler(request: Request, exc):
    """HTTP请求错误处理"""
    return Result.failed(exc.status_code, message=exc.detail)
```

### Response

```python
# 错误方法
def failed(status_code=200, message=failed_response.message):
    return JSONResponse(
        status_code=status_code or 200,
        content=FailedResponse(message=message).model_dump()
    )

# 错误内容
class FailedResponse(BaseResponse):
   code: int = -1
   data: dict = {}
   message: str = "failed"
```