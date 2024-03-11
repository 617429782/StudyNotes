表操作
  创建表
    CREATE TABLE `skyriver_vscode_report_0419` (
      `id` int(20) NOT NULL AUTO_INCREMENT COMMENT '自增主键',
      `misname` varchar(20) DEFAULT NULL COMMENT '本次打点的提交人',
      `report_timestamp` varchar(20) DEFAULT NULL COMMENT '本次打点时，灵犀统计的时间戳',
      `changedFileList` varchar(5000) DEFAULT NULL COMMENT '文件变动列表',
      PRIMARY KEY (`id`)
    )
    ENGINE=InnoDB AUTO_INCREMENT=2147483647 DEFAULT CHARSET=utf8mb4

  新增列
    ALTER TABLE `table_name` ADD COLUMN `col_name` VARCHAR(15);
    ALTER TABLE `table_name` ADD COLUMN `col_name` VARCHAR(15) AFTER `col_name`;

  删除列
    ALTER TABLE `table_name` DROP COLUMN `col_name`;

  重命名列
    ALTER TABLE `table_name` CHANGE COLUMN `old_col_name` `new_col_name` VARCHAR(25);
  
  修改行
    UPDATE 表名称 SET 列名称 = 新值 WHERE 列名称 = 某值


统计某一列中，各个值出现的次数
  select `col_name`, count(*) from `table_name` group by `col_name`

  select gitName, sum(status = 1) as '上线的页面数', sum(status = 0) as '开发中的页面数' from skyriver_page_detail group by gitName

统计今年新上线的页面数
  select * from skyriver_page_detail where publishTimestamp >= 1640966400000 and status = 1

模版选用率
  select * from log_template WHERE reportTimestamp >= 1671120000000 ORDER by reportTimestamp

去除重复行 misname、url、report_env 都相同的行合并成一条
  select * from dwtest.log_editUV group by misname,url,report_env