### sql.py

```python
from sqlalchemy import create_engine
from sqlalchemy.engine import Engine
from sqlalchemy.orm import sessionmaker, scoped_session, Session
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.exc import SQLAlchemyError


class MySQL:
    def __init__(self, host: str, port: int, user: str, password: str, database: str) -> None:
        self.__DATABASE_URL = f"mysql+pymysql://{user}:{password}@{host}:{port}/{database}?charset=utf8"
        self.__engine = self.__create_engine()

    def __create_engine(self) -> Engine:

        try:
            engin = create_engine(
                self.__DATABASE_URL,
                pool_size=5,  # 连接池大小
                max_overflow=5,  # 超过连接池大小外最多创建的连接
                pool_recycle=3600,  # 设置连接的最大空闲时间为 1 小时
                pool_timeout=15,  # 等待连接池中连接的最大时间
                pool_pre_ping=True,  # 在每次从连接池中获取连接时先发送一个简单的查询
                pool_reset_on_return="rollback",  # 把连接放回连接池之前默认要执行的操作
            )
            return engin
        except SQLAlchemyError as e:
            print(f"数据库引擎创建失败！\n{e}")
            raise

    def create_session(self) -> scoped_session[Session]:
        try:
            _session = sessionmaker(
                bind=self.__engine,
                autocommit=False,
                autoflush=True,
                expire_on_commit=False,
            )
            return scoped_session(_session)
        except SQLAlchemyError as e:
            print(f"数据库会话创建失败！\n{e}")
            raise

    @staticmethod
    def get_base():
        return declarative_base()
```



### __init__.py

```python
from .sql import MySQL
from core.config import settings
from sqlalchemy.sql import select
from sqlalchemy.exc import SQLAlchemyError

_mysql = MySQL(
    host=settings.DB_HOST,
    port=settings.DB_PORT,
    user=settings.DB_USER,
    password="Jch1063831321",
    database="anda_test",
)

SessionLocal = _mysql.create_session()
Base = _mysql.get_base()


def init_db():
    # 数据库首次连接
    db = SessionLocal()
    try:
        db.execute(select(1))
        print("MySQL数据库连接成功")
    except SQLAlchemyError as e:
        print(f"MySQL数据库连接失败: {e}")
    finally:
        db.close()
```

init_db放置在event_register中，在项目启动时随event事件启动