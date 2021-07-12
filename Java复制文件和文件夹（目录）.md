使用 	I/O 实现复制文件及文件夹

```java
/**
 * 用缓存的方式，复制文件
 *
 * @param src 源文件目录
 * @param des 输出文件目录
 */
public static void copyFile(String src, String des) {
    File srcFile = new File(src);
    if (!srcFile.exists()) {
        throw new RuntimeException(srcFile.getName() + "不存在");
    } else {
        File desFile = new File(des);
        try {
            //创建目标文件
            desFile.getParentFile().mkdirs();
            desFile.createNewFile();
        } catch (IOException e) {
            e.printStackTrace();
        }
        //缓存内容
        byte[] bufferContent = new byte[1024];
        try (
                FileInputStream fis = new FileInputStream(srcFile);
                FileOutputStream fos = new FileOutputStream(desFile)
        ) {
            //the total number of bytes read into the buffer
            int readLength = fis.read(bufferContent);

            //当读取的字符长度为 -1 时，说明没有可以读取的了
            while (readLength != -1) {
                fos.write(bufferContent, 0, readLength);
                fos.flush();
                readLength = fis.read(bufferContent);
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

/**
 * 复制文件夹（包括其文件）
 * @param srcPath 原目录
 * @param desPath 复制到的目录
 */
public static void copyFolder(String srcPath, String desPath) {
    File srcFolder = new File(srcPath);
    //如果原目录是一个 directory（包含了其存在的含义），就继续
    if (srcFolder.isDirectory()) {
        File desFolder = new File(desPath);

        //如果desFolder.isFile()，就说明它存在，且为文件
        //如果目标文件夹不是文件，说明它不存在，或者是一个目录，则继续
        if (!desFolder.isFile()) {
            desFolder.mkdirs();//不管怎么样，都可以先创建目录

            File[] files = srcFolder.listFiles();
            for (File file : files) {
                //新名字，位于原目录下……
                String newDesName = desFolder + File.separator + file.getName();

                if (file.isFile()) {
                    //如果是文件，调用文件的复制方法
                    copyFile(file.getAbsolutePath(), newDesName);
                }
                if (file.isDirectory()) {
                    //如果是文件夹，则递归处理
                    copyFolder(file.getAbsolutePath(), newDesName);
                }
            }
        }
    }
}
```