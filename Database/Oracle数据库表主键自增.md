# Oracle数据库表主键自增

对于表 `SOME_TABLE` 的主键 `ID` ，为其创建一个序列 `SEQ_SOME_TABLE_ID` ：

```sql
DROP SEQUENCE "SEQ_SOME_TABLE_ID";
CREATE SEQUENCE "SEQ_SOME_TABLE_ID"
 INCREMENT BY 1 START WITH 1
 NOMAXVALUE
 NOCYCLE
 CACHE 20;
```

再为该表添加一个触发器 `TRI_SOME_TABLE_INSERT` ：

```sql
CREATE OR REPLACE TRIGGER "TRI_SOME_TABLE_INSERT" BEFORE INSERT ON "SOME_TABLE" REFERENCING FOR EACH ROW 
BEGIN
	SELECT
		SEQ_SOME_TABLE_ID.nextval INTO :new.ID
	FROM
		dual ;
END ;
```

