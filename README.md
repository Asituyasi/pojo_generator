# Java Pojo Generator

can generator java pojo easyly.There is src code all

```
import java.io.*;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

/**
 * desc : pojo generator
 * Created by tiantian on 2018/11/8
 */
public class PojoSourceFileGerator {
    
    public static String driver = "com.mysql.jdbc.Driver";
    public static String url = "jdbc:mysql://localhost:3306/hrmanager?useSSL=false&useUnicode=true&autoReconnect=true&characterEncoding=utf8";
    public static String user = "root";
    public static String psd = "root";
    public static String dataBaseTableName = "user";
    public static String className = null;
    public static String filePath = "src/test/java/";
    
    public static void main(String[] args) throws SQLException, ClassNotFoundException {
        initialDBDriver();
        Connection connection = getConnection();
        DatabaseMetaData metaData = connection.getMetaData();
        Table table = obtainTable(metaData);

        SimpleGenerator simpleGenerator = new SimpleGenerator(table);
        String generatedCode = simpleGenerator.generateCode();
        //System.out.println(generatedCode);
        
        generateSrcFile(generatedCode);

        connection.close();
    }
    
    public static void generateSrcFile(String src) {
        File file = new File(filePath + className + ".java");
        if (!file.getParentFile().exists()) {
            file.getParentFile().mkdirs();
        }
        if (!file.exists()) {
            try {
                file.createNewFile();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        System.out.println(file.getAbsolutePath());

        FileOutputStream os = null;
        try {
            os = new FileOutputStream(file);
            OutputStreamWriter writer = new OutputStreamWriter(os, "utf-8");
            writer.write(src);
            writer.flush();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                os.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    
    static Table obtainTable(DatabaseMetaData metaData) throws SQLException {
        ResultSet columnSet = metaData.getColumns(null, null, dataBaseTableName, null);

        Table table = new Table();
        table.setName(dataBaseTableName);
        List<Column> columns = new ArrayList<>();
        table.setColumns(columns);
        while (columnSet.next()) {
            String columnName = columnSet.getString("COLUMN_NAME");
            String typeName = columnSet.getString("TYPE_NAME");
            String comment = columnSet.getString("REMARKS");

            Column column = new Column();
            column.setName(columnName);
            column.setComment(comment);
            
            typeName = typeName.split(" ")[0];
            if (typeName.contains("CHAR")) {
                column.setType("String");
            } else if (typeName.equals("INT")) {
                column.setType("Integer");
            } else if (typeName.equals("SMALLINT")) {
                column.setType("Short");
            } else if (typeName.equals("BIGINT")) {
                column.setType("Long");
            } else if (typeName.equals("FLOAT")) {
                column.setType("Float");
            } else if (typeName.equals("TIMESTAMP") || typeName.equals("DATE")) {
                column.setType("Date");
            } 

            columns.add(column);
        }
        
        return table;
    }
    
    static void initialDBDriver() throws ClassNotFoundException {
        Class.forName(driver);
    }
    
    public static Connection getConnection() {
        Connection connection = null;
        try {
            connection = DriverManager.getConnection(url, user, psd);
        } catch (SQLException e) {
            e.printStackTrace();
        }
        
        return connection;
    }
    
    private static class SimpleGenerator {
        private Table table;
        private String packageStr = "package empty;";
        private String publcClass = "public class ";

        public SimpleGenerator(Table table) {
            this.table = table;
        }

        public String generateCode() {
            StringBuilder builder = new StringBuilder();

            //builder.append(packageStr);
            builder.append("\n\n");
            builder.append(publcClass);
            className = table.getName();
            className = className.replaceFirst("\\w", className.substring(0, 1).toUpperCase());
            builder.append(className + " {\n\n");

            builder.append(generateProperty(false));
            builder.append(generateMethod());
            builder.append("}");
            return builder.toString();
        }

        private String generateMethod() {
            StringBuilder builder = new StringBuilder();

            for (Column column : table.getColumns()) {
                String type = column.getType();
                String columnName = column.getName();
                builder.append(generateGetMethod(columnName, type));
                builder.append(generateSetMethod(columnName, type));
            }
            
            return builder.toString();
        }

        private String generateSetMethod(String columnName, String type) {
            String headChar = columnName.substring(0, 1);
            String colName;
            if (columnName.split("_").length > 1) {
                colName = trimAndToCamelStyle(columnName);
                colName = colName.replaceFirst("\\w", headChar.toUpperCase());
            } else {
                colName = columnName.replaceFirst("\\w", headChar.toUpperCase());
            }
            
            String methodName = "";
            methodName += "\tpublic void ";
            String returnName = colName.replaceFirst("\\w", colName.substring(0,1).toLowerCase());
            methodName += "set" + colName + "(" +type +" " +returnName +") ";
            methodName += "{ this." + returnName + " = " + returnName +"; }" + "\n\n";

            return methodName;
        }

        private String generateGetMethod(String columnName, String type) {
            String colName;
            String headChar = columnName.substring(0, 1);
            // 下划线风格时
            if (columnName.split("_").length > 1) {
                colName = trimAndToCamelStyle(columnName);
                colName = colName.replaceFirst("\\w", headChar.toUpperCase());
            } else {
                colName = columnName.replaceFirst("\\w", headChar.toUpperCase());
            }
            
            String methodName = "";
            methodName += "\tpublic " + type;
            methodName += " get" + colName + "() ";
            String returnName = colName.replaceFirst("\\w", colName.substring(0,1).toLowerCase());
            methodName += "{ return " +  "this." + returnName + "; }" + "\n\n";
            
            return methodName;
        }
        
        private String generateProperty(boolean isIgnoreCommen) {
            StringBuilder builder = new StringBuilder();
            for (Column column : table.getColumns()) {
                String name = trimAndToCamelStyle(column.getName());
                String type = column.getType();
                builder.append("\tprivate " + type + " ");
                builder.append(name);
                builder.append(";");

                if (!isIgnoreCommen && !column.getComment().equals("")) {
                    builder.append(" // ");
                    builder.append(column.getComment());
                }

                builder.append("\n\n");
            }

            return builder.toString();
        }
        
    }
    
    private static String trimAndToCamelStyle(String string) {
        String result = "";
        if (string.split("_").length > 1) {
            String[] array = string.split("_");
            for (int i = 0; i < array.length; i++) {
                if (i > 0) {
                    array[i] = array[i].substring(0, 1).toUpperCase() 
                            + array[i].substring(1, array[i].length());
                }
                result += array[i];
            }
        } else {
            result = string;
        }
        
        return result;
    }
    
    private static class Table {
        private String name;
        private List<Column> columns;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public List<Column> getColumns() {
            return columns;
        }

        public void setColumns(List<Column> columns) {
            this.columns = columns;
        }

        @Override
        public String toString() {
            StringBuilder builder = new StringBuilder();
            for (Column column : getColumns()) {
                builder.append("{");
                builder.append(column.getName());
                builder.append("} ");
            }
            return "[" + getName() + "]:" + builder.toString();
        }
    }
    
    private static class Column {
        private String name;
        private String type;
        private String comment;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public String getType() {
            return type;
        }

        public void setType(String type) {
            this.type = type;
        }

        public String getComment() {
            return comment;
        }

        public void setComment(String comment) {
            this.comment = comment;
        }
    }
    
}
```
