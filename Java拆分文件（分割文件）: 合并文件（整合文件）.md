使用 I/O 内容，分解和合并文件。
注意，可以使用 DataOutputStream 和 DataInputStream，以及缓存等操作来改进。

```java
/**
 * 用于合并文件。
 * 其实还可以默认原文件和子文件在同一文件夹内，按照子文件的个数来合并文件，即：
 * mergeFile(String originFileName, int numOfFile, String desFileName)
 * 
 * @param desFileName 合并后的文件名
 * @param files 需要合并的文件
 */
public static void mergeFile(String desFileName, File... files) {
    File desFile = new File(desFileName);
    desFile.getParentFile().mkdirs();
    try {
        desFile.createNewFile();
    } catch (IOException e) {
        e.printStackTrace();
    }

    try (FileOutputStream fos = new FileOutputStream(desFile)) {
        for (File file : files) {
            if (!file.exists()) {
                throw new FileNotFoundException("无法读取" + file.getName());
            } else {
                //由于下面这个输入流在输出流的 try catch 内，所以不用再排除异常了
                FileInputStream fis = new FileInputStream(file);
                //读取碎片子文件的内容到 Java 内存中
                byte[] fileContent = new byte[(int) file.length()];
                fis.read(fileContent);
                fis.close();//记得这里要 close

                //写入新文件的硬盘中
                fos.write(fileContent);
                fos.flush();
            }
        }

    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }

}

/**
 * 将一个文件，划分为几份
 *
 * @param srcFile 原始文件
 * @param fileNum 分为多少份
 */
public static void splitFile(File srcFile, int fileNum) {

    if (srcFile.length() == 0 || !srcFile.exists()) {
        throw new RuntimeException("文件无法分割");
    } else {
        //用于存放文件内容（使用 byte 就足够读取数据了）
        byte[] fileContent = new byte[(int) srcFile.length()];

        //将内容加载到 Java 程序内存
        try (FileInputStream fis = new FileInputStream(srcFile)) {
            //将内容放入 fileContent 容器中
            fis.read(fileContent);
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }

        //使用 Math.ceil 计算每一份新文件的大小/长度
        int eachSize = (int) Math.ceil(srcFile.length() / (double) fileNum);

        for (int i = 0; i < fileNum; i++) {
            //如果不是最后一个文件
            if (i != fileNum - 1) {
                //获取该文件部分的内容
                byte[] eachFileCon = Arrays.copyOfRange(fileContent, i * eachSize
                        , (i + 1) * eachSize);
                //生产一份新的小文件
                generateEachFile(srcFile.getAbsolutePath(), i, eachFileCon);

            } else { //如果是最后一个文件……
                byte[] eachFileCon = Arrays.copyOfRange(fileContent, i * eachSize
                        , fileContent.length);
                generateEachFile(srcFile.getAbsolutePath(), i, eachFileCon);
            }
        }
    }
}

/**
 * 用于生成新的小文件，并输出到硬盘
 * 注意：（1）这里考虑了有后缀的情况 （2）用 .lastIndexOf() 应该也可以解决问题
 * @param originFileName 原始文件路径+文件名
 * @param num            编号
 * @param content        要存入该文件的内容
 */
private static void generateEachFile(String originFileName, int num, byte[] content) {
    //使用 split() 时，如果以"点"为分割，就要使用「Pattern.quote(".")」或者「"\\."」
    String[] split = originFileName.split("\\.");

    //用第一个点前面的字符串作为"基础"，然后在下面的代码中，在它后面加上"点"
    StringBuilder fileName = new StringBuilder(split[0]);

    //获取新文件名（新文件名为：原文件名-num.后缀）
    for (int i = 1; i < split.length; i++) {
        if (i != split.length - 1) {
            //在前面加上"点"，而不是在后面加上，因为这样可以保证编号之前没有"点"
            fileName.append("." + split[i]);
        } else {
            //加上编号和 .后缀
            fileName.append("-" + num + "." + split[i]);
        }
    }

    //读取新文件
    File file = new File(fileName.toString());

    //加入到内存，并输出
    try (FileOutputStream fos = new FileOutputStream(file)) {
        file.createNewFile();
        fos.write(content);
        fos.flush();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```