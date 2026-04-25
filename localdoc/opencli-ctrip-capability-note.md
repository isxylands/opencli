# `opencli ctrip` 能力边界说明

## 结论

当前仓库里的 `ctrip` 命令**不能**查询国内航班和票价，也**不能**按“出发时间、到达机场、每小时最低价”这类条件做航班筛选。

它现在只做一件事：

- 通过 `https://m.ctrip.com/restapi/soa2/21881/json/gaHotelSearchEngine`
- 查询携程的目的地、景区和酒店联想结果

所以像下面这种需求，现有实现并不支持：

- 北京首都 -> 成都双流
- 后天 18:00 以后
- 每小时最低价
- 国内航班列表

## 依据

- `clis/ctrip/search.ts`
- `clis/ctrip/search.test.ts`
- `docs/adapters/browser/ctrip.md`
- `skills/opencli-usage/browser.md`

关键特征很明确：

- 命令只有 `search <query>` 和 `--limit`
- 请求目标是 `gaHotelSearchEngine`
- 输出字段只有 `name/type/score/price/url`
- 测试只覆盖目的地和酒店联想结果

## 误区提醒

不要把“结果里出现 `price` 字段”误解成“能查机票票价”。这里的 `price` 只是联想结果里的展示字段，和航班查询没有关系。

## 如果要支持航班查询

更合理的做法不是硬扩展当前 `search`，而是新增独立命令，例如：

- `opencli ctrip flight-search`
- 参数至少应包含：`from`、`to`、`date`、`depart-after`
- 如果要支持“每小时最低价”，还需要对返回航班按小时分组，再做最小值聚合

实现上建议先确认携程是否有稳定可调用的航班接口，或者直接走浏览器页面抓取结果页，而不是复用现有酒店联想接口。
