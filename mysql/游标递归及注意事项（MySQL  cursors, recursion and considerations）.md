一.MYSQL存储过程递归及游标使用
------
1.根据tree结构，找到最顶层节点下面所有底层节点(中间或许有很多层)。
```sql
DROP TABLE IF EXISTS `tree` ;
CREATE TABLE `tree` (
    `id`  BIGINT(20) NOT NULL AUTO_INCREMENT,
    `parentId`  BIGINT(20) NULL COMMENT '父id', 
    `type`  INT(3) NULL COMMENT '节点类型，1:最顶层节点；10:最底层节点', 
     PRIMARY KEY (`id`)
)ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
2.创建temp_tree临时表来存储最顶层和最底层节点的关系。
```sql
DROP TABLE IF EXISTS `temp_tree` ;
CREATE TABLE `temp_tree` (
    `id`  BIGINT(20) NOT NULL AUTO_INCREMENT, 
    `treeId` BIGINT(20) COMMENT '最底层节点id', 
    `rootTreeId` BIGINT(20) COMMENT '最顶层节点id', 
    PRIMARY KEY (`id`)
)ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT '最顶层和最底层节点的关系临时表';
```
3.创建copytreeToTempTree存储过程，并递归找子节点。  
(1)查询出所有root节点。  
(2)设置递归深度(这里设置了10)。  
(3)用游标循环调用findChildren()。  
(4)递归完后关闭游标，结束。  
``` sql
DROP PROCEDURE if EXISTS `copytreeToTempTree`;
DELIMITER ;;
CREATE PROCEDURE copytreeToTempTree()
BEGIN
	DECLARE finished INT DEFAULT 0;
	DECLARE nodeId VARCHAR(255);
	DECLARE cur CURSOR FOR SELECT id FROM `tree` WHERE type = 1;
	DECLARE CONTINUE HANDLER FOR not found set finished = 1;
	OPEN cur;
	FETCH cur into nodeId;
	set max_sp_recursion_depth  = 10;
	WHILE finished = 0 DO
		call findChildren(nodeId, nodeId);
		FETCH cur into nodeId;
	END WHILE ;
	CLOSE cur;
END ;;
DELIMITER ;
```
4.利用游标，遍历节点，假如是最底层节点，把当前节点及root节点插入到temp_tree。
  否则在调用findChildren，直到找到最底层节点。
```sql
DROP PROCEDURE if EXISTS `findChildren`;
DELIMITER ;;
CREATE PROCEDURE findChildren(in pnodeId VARCHAR (255), in rootNodeId VARCHAR(255))
BEGIN
	DECLARE finished INT DEFAULT 0;
	DECLARE nodeId VARCHAR(255);
	DECLARE typel  VARCHAR(255);
	DECLARE cur CURSOR FOR SELECT id, type from `tree` WHERE `parentId` = pnodeId;
	DECLARE CONTINUE HANDLER FOR not found set finished = 1;
	OPEN cur;
	FETCH cur into nodeId, typel;
	WHILE finished = 0 DO
		IF typel= 10 THEN
			INSERT INTO temp_tree VALUES(nodeId,rootNodeId);
		ELSE
			call findChildren(nodeId,rootNodeId);
		END if;
		FETCH cur into nodeId, typel;
	END WHILE ;
	CLOSE cur;
END ;;
DELIMITER ;
```
5.调用copytreeToTempTree存储过程，把最底层节点和root节点插入temp_tree。
```sql
call copytreeToTempTree();
```
二.注意事项
-------
1.findChildren声明接收type的为什么是type1。  
  `因为游标里面是type，再用type接收不到值，一直是null.(暂不知实现原理)`  
2.报这个错”Thread stack overrun:  11552 bytes used of a 131072 byte stack, and 128000 bytes needed.  Use 'mysqld --thread_stack=#' to specify a bigger stack.“  
  `有可能是递归没跳出去，导致死循环。` 
