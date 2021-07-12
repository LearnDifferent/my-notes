搜索文件，找到相应的字符串，并打印出来。

```java
/**
 * 找到特定字符串，并打印出来
 *
 * @param file    被搜索的文件/文件夹
 * @param pattern 要寻找的字符
 */
public static void search(File file, String pattern) {
    if (file.isFile()) {
        findAndPrint(file, pattern);
    }
    if (file.isDirectory()) {
        File[] files = file.listFiles();
        for (File f : files) {
            search(f, pattern);
        }
    }
}

private static void findAndPrint(File file, String pattern) {
    try (
            FileInputStream fis = new FileInputStream(file);
            InputStreamReader isr = new InputStreamReader(fis, Charset.forName("UTF-8"));
            BufferedReader br = new BufferedReader(isr)

    ) {
        String line;//读取的每一行数据
        boolean notFind = true;//是否还美找到 pattern 字符
        //如果 BufferedReader 的 readLine() 方法读不出数据，就会返回 null
        while ((line = br.readLine()) != null && notFind) {
            if (line.contains(pattern)) {
                notFind = true;
                //其实可以详细显示在哪一行，找到了该字符串，这里就简单写一下。
                System.out.println("找到含有「" + pattern + "」的文件：" + file.getAbsolutePath());
            }
        }

    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```