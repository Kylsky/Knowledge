# **MySQL练习(一)**

## 创建测试数据

```
/*
 Navicat Premium Data Transfer

 Source Server         : MySQL
 Source Server Type    : MySQL
 Source Server Version : 50562
 Source Host           : localhost:3306
 Source Schema         : test

 Target Server Type    : MySQL
 Target Server Version : 50562
 File Encoding         : 65001

 Date: 15/06/2019 11:21:30
*/

SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for dept
-- ----------------------------
DROP TABLE IF EXISTS `dept`;
CREATE TABLE `dept`  (
  `deptno` int(2) NOT NULL,
  `dname` varchar(15) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `loc` varchar(15) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  PRIMARY KEY (`deptno`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Compact;

-- ----------------------------
-- Records of dept
-- ----------------------------
INSERT INTO `dept` VALUES (10, 'ACCOUNTING', 'NewYork');
INSERT INTO `dept` VALUES (20, 'RESEARCH', 'Dallas');
INSERT INTO `dept` VALUES (30, 'SALES', 'Chicago');
INSERT INTO `dept` VALUES (40, 'OPERATIONS', 'Boston');

-- ----------------------------
-- Table structure for emp
-- ----------------------------
DROP TABLE IF EXISTS `emp`;
CREATE TABLE `emp`  (
  `empno` int(4) NOT NULL COMMENT '雇员编号',
  `ename` varchar(10) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `job` varchar(10) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `mgr` int(4) NULL DEFAULT NULL,
  `hiredate` date NULL DEFAULT NULL,
  `sal` decimal(7, 0) NULL DEFAULT NULL,
  `comm` decimal(7, 0) NULL DEFAULT NULL,
  `deptno` int(2) NULL DEFAULT NULL,
  PRIMARY KEY (`empno`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Compact;

-- ----------------------------
-- Records of emp
-- ----------------------------
INSERT INTO `emp` VALUES (7369, 'SMITH', 'CLERK', 7902, '1980-12-17', 800, NULL, 20);
INSERT INTO `emp` VALUES (7499, 'ALLEN', 'SALESMAN', 7698, '1981-02-20', 1600, 300, 30);
INSERT INTO `emp` VALUES (7521, 'WARD', 'SALESMAN', 7698, '1981-02-22', 1250, 500, 30);
INSERT INTO `emp` VALUES (7566, 'JONES', 'MANAGER', 7839, '1981-04-02', 2975, NULL, 20);
INSERT INTO `emp` VALUES (7654, 'MARTIN', 'SALESMAN', 7698, '1981-09-28', 1250, 1400, 30);
INSERT INTO `emp` VALUES (7698, 'BLAKE', 'MANAGER', 7839, '1981-05-01', 2850, NULL, 30);
INSERT INTO `emp` VALUES (7782, 'CLARK', 'MANAGER', 7839, '1981-06-09', 2450, NULL, 10);
INSERT INTO `emp` VALUES (7788, 'SCOTT', 'ANALYST', 7566, '1987-07-13', 3000, NULL, 20);
INSERT INTO `emp` VALUES (7839, 'KING', 'PRESIDENT', NULL, '1981-11-17', 5000, NULL, 10);
INSERT INTO `emp` VALUES (7844, 'TURNER', 'SALESMAN', 7698, '1981-09-08', 1500, 0, 30);
INSERT INTO `emp` VALUES (7876, 'ADAMS', 'CLERK', 7788, '1987-07-13', 1100, NULL, 20);
INSERT INTO `emp` VALUES (7900, 'JAMES', 'CLERK', 7698, '1981-12-03', 950, NULL, 30);
INSERT INTO `emp` VALUES (7902, 'FORD', 'ANALYST', 7566, '1981-12-03', 3000, NULL, 20);
INSERT INTO `emp` VALUES (7934, 'MILLER', 'CLERK', 7782, '1982-01-23', 1300, NULL, 10);

-- ----------------------------
-- Table structure for salgrade
-- ----------------------------
DROP TABLE IF EXISTS `salgrade`;
CREATE TABLE `salgrade`  (
  `grade` int(7) NULL DEFAULT NULL,
  `losal` int(7) NULL DEFAULT NULL,
  `hisal` int(7) NULL DEFAULT NULL
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Compact;

-- ----------------------------
-- Records of salgrade
-- ----------------------------
INSERT INTO `salgrade` VALUES (1, 700, 1200);
INSERT INTO `salgrade` VALUES (2, 1201, 1400);
INSERT INTO `salgrade` VALUES (3, 1401, 2000);
INSERT INTO `salgrade` VALUES (4, 2001, 3000);
INSERT INTO `salgrade` VALUES (5, 3001, 9999);

SET FOREIGN_KEY_CHECKS = 1;
```



## 1.部门平均薪水的等级

### 思路：

先算出部门平均薪水，再按照计算得出的薪水做出等级判断

### 答案：

```
SELECT deptno,avg_sal,grade
FROM salgrade
JOIN
(SELECT AVG(sal) AS avg_sal,deptno 
FROM emp
GROUP BY deptno) AS sal
ON
sal.avg_sal BETWEEN salgrade.losal AND salgrade.hisal
```



## 2.部门平均的薪水等级

### 思路：

算出每一个部门每一个人的薪水等级，然后将薪水等级做平均值计算

### 答案：

```
SELECT deptno,AVG(grade) FROM
(SELECT deptno,sal,grade FROM emp
JOIN salgrade
ON emp.sal BETWEEN salgrade.losal AND salgrade.hisal)AS t
GROUP BY deptno
```



## 3.哪些人是经理

### 思路：

job为MANAGER的是经理，但是mgr作为他的对应上级也是经理，所以根据mgr来筛选

### 答案：

```
SELECT ename FROM emp 
WHERE empno IN
(SELECT DISTINCT mgr FROM emp)
```



## 4.不用组函数求最高薪水

### 思路:

**思路一：**可以用group by(sal)+order by(sal) desc然后limit 1实现

**思路二：**将表进行内连接，匹配条件为左表sal小于右表sal，则可以找出所有非最高薪水的所有薪水，再利用not in找到最高薪水

### 答案：

```
SELECT sal FROM emp
GROUP BY sal
ORDER BY sal DESC
LIMIT 1
```

or

```
SELECT sal FROM emp
WHERE sal NOT IN
(SELECT DISTINCT t1.sal 
FROM emp AS t1
JOIN emp AS t2
ON
t1.sal < t2.sal)
```



## 5.平均薪水最高的部门编号与名称

### 思路：

1.计算每个部门的平均薪水;

2.计算最高平均薪水的部门

3.根据最高平均薪水找到部门编号

4.根据部门编号通过表连接找到名称

### 答案：

```
SELECT t3.deptno,dname
FROM dept JOIN 
(SELECT deptno,avg_sal FROM 
(SELECT deptno,AVG(sal) AS avg_sal
FROM emp
GROUP BY deptno) AS t1 
WHERE avg_sal=
(SELECT MAX(avg_sal) AS max_avg_sal FROM
	( SELECT deptno, AVG( sal ) AS avg_sal FROM emp GROUP BY deptno ) AS t2 ))AS t3
	WHERE t3.deptno = dept.deptno
```



## 6.求平均薪水最高的部门的部门编号

### 思路：

与上题类似，不赘述了。

### 答案：

```
SELECT deptno FROM 
(SELECT deptno,AVG(sal) AS avg_sal
FROM emp
GROUP BY deptno) AS t1 
WHERE avg_sal=
(SELECT MAX(avg_sal) AS max_avg_sal FROM
	( SELECT deptno, AVG( sal ) AS avg_sal FROM emp GROUP BY deptno ) AS t2 )
```

or

```
select deptno, avg_sal from
	(select avg(sal) avg_sal, deptno from emp group by deptno) as t2
	where avg_sal = 
	(select max(avg(sal)) from emp group by deptno);
//嵌套组函数再Mysql5.5中不支持
```



## 7.求平均薪水的等级最低的部门的部门名称

### 思路：

1.求出各部门平均薪水的等级

2.求出最低的平均薪水的等级的部门编号

3.根据部门编号通过表连接找到部门名称

### 答案：

```

SELECT dept.deptno,dname
FROM dept JOIN
(SELECT * FROM 
(SELECT deptno,grade
FROM salgrade
JOIN
(SELECT AVG(sal) AS avg_sal,deptno 
FROM emp
GROUP BY deptno) AS sal
ON
sal.avg_sal BETWEEN salgrade.losal AND salgrade.hisal) AS t2
WHERE grade = 
(SELECT MIN(grade) AS min_grade FROM
(SELECT deptno,grade
FROM salgrade
JOIN
(SELECT AVG(sal) AS avg_sal,deptno 
FROM emp
GROUP BY deptno) AS sal
ON
sal.avg_sal BETWEEN salgrade.losal AND salgrade.hisal) AS t1))AS t3
ON t3.deptno = dept.deptno

```



## 8.比普通员工的最高薪水还要高的经理人名称

### 思路：

1.找到普通员工，即找到经理并排除

2.找到普通员工的最高薪水

3.找到比普通员工最高薪水还要高的经理人名称

### 答案：

```
SELECT ename FROM emp 
WHERE empno IN
(SELECT DISTINCT mgr FROM emp)
AND sal >
(SELECT MAX(sal) FROM 
(SELECT empno,sal FROM emp 
WHERE empno NOT IN
(SELECT empno FROM emp 
WHERE empno IN
(SELECT DISTINCT mgr FROM emp)))AS t1)
```

