### Working Process 


FLINK SQL 执行过程的深度剖析

从一条DDL语句进入开始讲起
```sql
CREATE TABLE json_source (
    id            BIGINT,
    name          STRING,
    date1         DATE,
    date2         DATE,
    time1         MAP<STRING,TIME>,
    time2         TIME,
    timestamp1    TIMESTAMP(3),
    timestamp2    TIMESTAMP(3),
    `map`         MAP<STRING,BIGINT>,
    map2map       MAP<STRING,MAP<STRING,INTEGER>>,
    proctime      as PROCTIME()
 ) WITH (
    'connector.type'                          = 'kafka',
    'connector.topic'                         = 'json_demo',
    'connector.properties.bootstrap.servers'  = 'localhost:9092',
    'connector.properties.group.id'           = 'testGroup',
    'connector.version'                       = 'universal',
    'format.type'                             = 'json',
    'connector.startup-mode'                  = 'latest-offset'
);
```
在命令行输入代码之后。

下面的代码会循环从命令行读取用户的输入并解析执行。读入的触发条件是";" 之后点击回车键。
当触发读取之后会将所有内容读入并赋值给`final String line;`,之后不可再修改（在这个测试中`line`的内容就是上面输入的SQL内容）。

```
// print welcome
terminal.writer().append(CliStrings.MESSAGE_WELCOME);
// begin reading loop
while (isRunning) {
	// make some space to previous command
	terminal.writer().append("\n");
	terminal.flush();
	final String line;
	try {
		line = lineReader.readLine(prompt, null, (MaskingCallback) null, null);
	} catch (UserInterruptException e) {
		// user cancelled line with Ctrl+C
		continue;
	} catch (EndOfFileException | IOError e) {
		// user cancelled application with Ctrl+D or kill
		break;
	} catch (Throwable t) {
		throw new SqlClientException("Could not read from command line.", t);
	}
	if (line == null) {
		continue;
	}
	final Optional<SqlCommandCall> cmdCall = parseCommand(line);
	cmdCall.ifPresent(this::callCommand);
}

```

紧接着,以`line`为入参数进入`parseCommand(line)` 方法。这个方法会调用`SqlCommandParser.parse()`方法返回一个`SqlCommandCall`对象。

```
private Optional<SqlCommandCall> parseCommand(String line) {
    final SqlCommandCall parsedLine;
    try {
        parsedLine = SqlCommandParser.parse(executor.getSqlParser(sessionId), line);
    } catch (SqlExecutionException e) {
        printExecutionException(e);
        return Optional.empty();
    }
    return Optional.of(parsedLine);
}
```

我们继续进入`SqlCommandParser.parse()`方法一探究竟。在这个方法中首先会调用`String.trim()`方法去除sql文本两端的空格和末端的`;`。
然后先用正则匹配的方式对sql进行判断。

```
public static SqlCommandCall parse(Parser sqlParser, String stmt) {
	// normalize
	stmt = stmt.trim();
	// remove ';' at the end
	if (stmt.endsWith(";")) {
		stmt = stmt.substring(0, stmt.length() - 1).trim();
	}

	// parse statement via regex matching first
	Optional<SqlCommandCall> callOpt = parseByRegexMatching(stmt);
	if (callOpt.isPresent()) {
		return callOpt.get();
	} else {
		return parseBySqlParser(sqlParser, stmt);
	}
}
```
附
```
/**
 * Supported SQL commands.
 */
enum SqlCommand {
	QUIT(
		"(QUIT|EXIT)",
		NO_OPERANDS),

	CLEAR(
		"CLEAR",
		NO_OPERANDS),

	HELP(
		"HELP",
		NO_OPERANDS),

	SHOW_CATALOGS,

	SHOW_DATABASES,

	SHOW_TABLES,

	SHOW_FUNCTIONS,

	// FLINK-17396
	SHOW_MODULES(
		"SHOW\\s+MODULES",
		NO_OPERANDS),

	USE_CATALOG,

	USE,

	CREATE_CATALOG,

	DROP_CATALOG,

	DESC(
		"DESC\\s+(.*)",
		SINGLE_OPERAND),

	DESCRIBE,

	// supports both `explain xx` and `explain plan for xx` now
	// TODO should keep `explain xx` ?
	// only match "EXPLAIN SELECT xx" and "EXPLAIN INSERT xx" here
	// "EXPLAIN PLAN FOR xx" should be parsed via sql parser
	EXPLAIN(
		"EXPLAIN\\s+(SELECT|INSERT)\\s+(.*)",
		(operands) -> {
			return Optional.of(new String[] { operands[0], operands[1] });
		}),

	CREATE_DATABASE,

	DROP_DATABASE,

	ALTER_DATABASE,

	CREATE_TABLE,

	DROP_TABLE,

	ALTER_TABLE,

	CREATE_VIEW,

	DROP_VIEW,

	ALTER_VIEW,

	CREATE_FUNCTION,

	DROP_FUNCTION,

	ALTER_FUNCTION,

	SELECT,

	INSERT_INTO,

	INSERT_OVERWRITE,

	SET(
		"SET(\\s+(\\S+)\\s*=(.*))?", // whitespace is only ignored on the left side of '='
		(operands) -> {
			if (operands.length < 3) {
				return Optional.empty();
			} else if (operands[0] == null) {
				return Optional.of(new String[0]);
			}
			return Optional.of(new String[]{operands[1], operands[2]});
		}),

	RESET(
		"RESET",
		NO_OPERANDS),

	SOURCE(
		"SOURCE\\s+(.*)",
		SINGLE_OPERAND);

	public final @Nullable Pattern pattern;
	public final @Nullable Function<String[], Optional<String[]>> operandConverter;

	SqlCommand() {
		this.pattern = null;
		this.operandConverter = null;
	}
```

进入`parseBySqlParser(Parser sqlParser, String stmt)`方法。

```
private static SqlCommandCall parseBySqlParser(Parser sqlParser, String stmt) {
	List<Operation> operations;
	try {
		operations = sqlParser.parse(stmt);
	} catch (Throwable e) {
		throw new SqlExecutionException("Invalidate SQL statement.", e);
	}
	if (operations.size() != 1) {
		throw new SqlExecutionException("Only single statement is supported now.");
	}

	final SqlCommand cmd;
	String[] operands = new String[] { stmt };
	Operation operation = operations.get(0);
	if (operation instanceof CatalogSinkModifyOperation) {
		boolean overwrite = ((CatalogSinkModifyOperation) operation).isOverwrite();
		cmd = overwrite ? SqlCommand.INSERT_OVERWRITE : SqlCommand.INSERT_INTO;
	} else if (operation instanceof CreateTableOperation) {
		cmd = SqlCommand.CREATE_TABLE;
	} else if (operation instanceof DropTableOperation) {
		cmd = SqlCommand.DROP_TABLE;
	} else if (operation instanceof AlterTableOperation) {
		cmd = SqlCommand.ALTER_TABLE;
	} else if (operation instanceof CreateViewOperation) {
		cmd = SqlCommand.CREATE_VIEW;
	} else if (operation instanceof DropViewOperation) {
		cmd = SqlCommand.DROP_VIEW;
	} else if (operation instanceof AlterViewOperation) {
		cmd = SqlCommand.ALTER_VIEW;
	} else if (operation instanceof CreateDatabaseOperation) {
		cmd = SqlCommand.CREATE_DATABASE;
	} else if (operation instanceof DropDatabaseOperation) {
		cmd = SqlCommand.DROP_DATABASE;
	} else if (operation instanceof AlterDatabaseOperation) {
		cmd = SqlCommand.ALTER_DATABASE;
	} else if (operation instanceof CreateCatalogOperation) {
		cmd = SqlCommand.CREATE_CATALOG;
	} else if (operation instanceof DropCatalogOperation) {
		cmd = SqlCommand.DROP_CATALOG;
	} else if (operation instanceof UseCatalogOperation) {
		cmd = SqlCommand.USE_CATALOG;
		operands = new String[] { ((UseCatalogOperation) operation).getCatalogName() };
	} else if (operation instanceof UseDatabaseOperation) {
		cmd = SqlCommand.USE;
		operands = new String[] { ((UseDatabaseOperation) operation).getDatabaseName() };
	} else if (operation instanceof ShowCatalogsOperation) {
		cmd = SqlCommand.SHOW_CATALOGS;
		operands = new String[0];
	} else if (operation instanceof ShowDatabasesOperation) {
		cmd = SqlCommand.SHOW_DATABASES;
		operands = new String[0];
	} else if (operation instanceof ShowTablesOperation) {
		cmd = SqlCommand.SHOW_TABLES;
		operands = new String[0];
	} else if (operation instanceof ShowFunctionsOperation) {
		cmd = SqlCommand.SHOW_FUNCTIONS;
		operands = new String[0];
	} else if (operation instanceof CreateCatalogFunctionOperation ||
			operation instanceof CreateTempSystemFunctionOperation) {
		cmd = SqlCommand.CREATE_FUNCTION;
	} else if (operation instanceof DropCatalogFunctionOperation ||
			operation instanceof DropTempSystemFunctionOperation) {
		cmd = SqlCommand.DROP_FUNCTION;
	} else if (operation instanceof AlterCatalogFunctionOperation) {
		cmd = SqlCommand.ALTER_FUNCTION;
	} else if (operation instanceof ExplainOperation) {
		cmd = SqlCommand.EXPLAIN;
	} else if (operation instanceof DescribeTableOperation) {
		cmd = SqlCommand.DESCRIBE;
		operands = new String[] { ((DescribeTableOperation) operation).getSqlIdentifier().asSerializableString() };
	} else if (operation instanceof QueryOperation) {
		cmd = SqlCommand.SELECT;
	} else {
		throw new SqlExecutionException("Unknown operation: " + operation.asSummaryString());
	}

	return new SqlCommandCall(cmd, operands);
}

```
进入 blink 中的`parse(String statement)` 方法,使用`CalciteParser`对SQL进行解析得到`SqlNode`对象。
然后使用`SqlToOperationConverter.convert()` 对象将`SqlNode`对象转为`Operation`对象。

```$xslt
public List<Operation> parse(String statement) {
	CalciteParser parser = calciteParserSupplier.get();
	FlinkPlannerImpl planner = validatorSupplier.get();
	// parse the sql query
	SqlNode parsed = parser.parse(statement);

	Operation operation = SqlToOperationConverter.convert(planner, catalogManager, parsed)
		.orElseThrow(() -> new TableException("Unsupported query: " + statement));
	return Collections.singletonList(operation);
}
```

`SqlToOperationConverter.convert(FlinkPlannerImpl flinkPlanner,CatalogManager catalogManager,SqlNode sqlNode)` 方法如下

```$xslt
public static Optional<Operation> convert(
		FlinkPlannerImpl flinkPlanner,
		CatalogManager catalogManager,
		SqlNode sqlNode) {
	// validate the query
	final SqlNode validated = flinkPlanner.validate(sqlNode);
	SqlToOperationConverter converter = new SqlToOperationConverter(flinkPlanner, catalogManager);
	if (validated instanceof SqlCreateTable) {
		return Optional.of(converter.createTableConverter.convertCreateTable((SqlCreateTable) validated));
	} else if (validated instanceof SqlDropTable) {
		return Optional.of(converter.convertDropTable((SqlDropTable) validated));
	} else if (validated instanceof SqlAlterTable) {
		return Optional.of(converter.convertAlterTable((SqlAlterTable) validated));
	} else if (validated instanceof SqlAlterView) {
		return Optional.of(converter.convertAlterView((SqlAlterView) validated));
	} else if (validated instanceof SqlCreateFunction) {
		return Optional.of(converter.convertCreateFunction((SqlCreateFunction) validated));
	} else if (validated instanceof SqlAlterFunction) {
		return Optional.of(converter.convertAlterFunction((SqlAlterFunction) validated));
	} else if (validated instanceof SqlDropFunction) {
		return Optional.of(converter.convertDropFunction((SqlDropFunction) validated));
	} else if (validated instanceof RichSqlInsert) {
		return Optional.of(converter.convertSqlInsert((RichSqlInsert) validated));
	} else if (validated instanceof SqlUseCatalog) {
		return Optional.of(converter.convertUseCatalog((SqlUseCatalog) validated));
	} else if (validated instanceof SqlUseDatabase) {
		return Optional.of(converter.convertUseDatabase((SqlUseDatabase) validated));
	} else if (validated instanceof SqlCreateDatabase) {
		return Optional.of(converter.convertCreateDatabase((SqlCreateDatabase) validated));
	} else if (validated instanceof SqlDropDatabase) {
		return Optional.of(converter.convertDropDatabase((SqlDropDatabase) validated));
	} else if (validated instanceof SqlAlterDatabase) {
		return Optional.of(converter.convertAlterDatabase((SqlAlterDatabase) validated));
	} else if (validated instanceof SqlCreateCatalog) {
		return Optional.of(converter.convertCreateCatalog((SqlCreateCatalog) validated));
	} else if (validated instanceof SqlDropCatalog) {
		return Optional.of(converter.convertDropCatalog((SqlDropCatalog) validated));
	} else if (validated instanceof SqlShowCatalogs) {
		return Optional.of(converter.convertShowCatalogs((SqlShowCatalogs) validated));
	} else if (validated instanceof SqlShowDatabases) {
		return Optional.of(converter.convertShowDatabases((SqlShowDatabases) validated));
	} else if (validated instanceof SqlShowTables) {
		return Optional.of(converter.convertShowTables((SqlShowTables) validated));
	} else if (validated instanceof SqlShowFunctions) {
		return Optional.of(converter.convertShowFunctions((SqlShowFunctions) validated));
	} else if (validated instanceof SqlCreateView) {
		return Optional.of(converter.convertCreateView((SqlCreateView) validated));
	} else if (validated instanceof SqlDropView) {
		return Optional.of(converter.convertDropView((SqlDropView) validated));
	} else if (validated instanceof SqlShowViews) {
		return Optional.of(converter.convertShowViews((SqlShowViews) validated));
	} else if (validated instanceof SqlExplain) {
		return Optional.of(converter.convertExplain((SqlExplain) validated));
	} else if (validated instanceof SqlRichDescribeTable) {
		return Optional.of(converter.convertDescribeTable((SqlRichDescribeTable) validated));
	} else if (validated.getKind().belongsTo(SqlKind.QUERY)) {
		return Optional.of(converter.convertSqlQuery(validated));
	} else {
		return Optional.empty();
	}
}

```





