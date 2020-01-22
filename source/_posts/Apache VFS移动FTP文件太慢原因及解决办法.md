---
title: Apache VFS移动FTP文件太慢原因及解决办法
date: 2019-08-12 19:56:45
categories:
 - Java
tags:
 - Java
 - FTP
 - Apache VFS
---

项目中我们使用Apache VFS操作FTP服务器上得文件。但是最近发现，如果一个文件夹里面的文件特别多，移动这个文件夹里的文件就会特别慢。



于是，我就找了找原因。

<!-- more -->

`Apache VFS`移动文件是通过使用`FileSystemManager的resolveFile方法`获得`FileObject`，然后调用其`moveTo`方法来达到FTP文件移动的目的。

我们使用的`FileSystemManager`是默认的`DefaultFileSystemManager`，在操作FTP文件的时候会调用`AbstractOriginatingFileProvider`的`findFile`方法。

{% codeblock AbstractOriginatingFileProvider.findFile lang:java %}
protected FileObject findFile(final FileName name, final FileSystemOptions fileSystemOptions)
            throws FileSystemException {
    // Check in the cache for the file system
    final FileName rootName = getContext().getFileSystemManager().resolveName(name, FileName.ROOT_PATH);
    final FileSystem fs = getFileSystem(rootName, fileSystemOptions);
    // Locate the file
    // return fs.resolveFile(name.getPath());
    return fs.resolveFile(name);
}
{% endcodeblock %}

从这里可以看到会使用`FileName`获取到一个`FileSystem`，然后调用FlieSystem的resolveFile方法。这个FileName是从FTP的uri中解析出来的。FTP的uri（例如：[ftp://username:password@host](ftp://username:password@host):port/）如果username，password，host，port相同，这里取到的`FileSystem是同一个`。这里涉及到两个重要的类。

{% tabs class %}

<!-- tab FtpFileObject -->
{% codeblock FtpFileObject.class lang:java %}
protected FtpFileObject(final AbstractFileName name, final FtpFileSystem fileSystem, final FileName rootName)
            throws FileSystemException {
    super(name, fileSystem);
    final String relPath = UriParser.decode(rootName.getRelativeName(name));
    if (".".equals(relPath)) {
        // do not use the "." as path against the ftp-server
        // e.g. the uu.net ftp-server do a recursive listing then
        // this.relPath = UriParser.decode(rootName.getPath());
        // this.relPath = ".";
        this.relPath = null;
    } else {
        this.relPath = relPath;
    }
}
{% endcodeblock %}
从构造函数可以看出，并没有做太多事情，而且最关键的属性没有初始化
```
private FTPFile fileInfo;
```
<!-- endtab -->
<!-- tab FtpFileSystem -->
{% codeblock FtpFileSystem.class lang:java  %}
public void putClient(final FtpClient client) {
    // Save client for reuse if none is idle.
    if (!idleClient.compareAndSet(null, client)) {
        // An idle client is already present so close the connection.
        closeConnection(client);
    }
}
public FtpClient getClient() throws FileSystemException {
    FtpClient client = idleClient.getAndSet(null);
    if (client == null || !client.isConnected()) {
        client = createWrapper();
    }
    return client;
}
这个类就是对`FtpClient`进行了封装，操作FTP文件时会先调用`getClient()`，操作完成后再调用`putClient`。这个类使用`AtomicReference`来保持他只持有`一个`FtpClient，每次get的时候会置`null`，如果有`其他的`线程get，那么会创建一个`新的client`返回。在put的时候，如果这个类已经持有一个client了，就把put进来的client关掉。
{% endcodeblock %}
<!-- endtab -->

{% endtabs %}



既然是移动文件太慢，那就看看`AbstractFileObject`的`moveTo`方法

{% codeblock AbstractFileObject.class lang:java %}
@Override
public void moveTo(final FileObject destFile) throws FileSystemException {
    if (canRenameTo(destFile)) {
        if (!getParent().isWriteable()) {
            throw new FileSystemException("vfs.provider/rename-parent-read-only.error", getName(),getParent().getName());
        }
    } else {
        if (!isWriteable()) {
            throw new FileSystemException("vfs.provider/rename-read-only.error", getName());
        }
    }
    if (destFile.exists() && !isSameFile(destFile)) {
        destFile.deleteAll();
        // throw new FileSystemException("vfs.provider/rename-dest-exists.error", destFile.getName());
    }
    if (canRenameTo(destFile)) {
        // issue rename on same filesystem
        try {
            attach();
            // remember type to avoid attach
            final FileType srcType = getType();
            doRename(destFile); 
            FileObjectUtils.getAbstractFileObject(destFile).handleCreate(srcType);
            destFile.close(); // now the destFile is no longer imaginary. force reattach.
            handleDelete(); // fire delete-events. This file-object (src) is like deleted.
            } catch (final RuntimeException re) {
                throw re;
            } catch (final Exception exc) {
                throw new FileSystemException("vfs.provider/rename.error", exc, getName(), destFile.getName());
            }
        } else {
            // different fs - do the copy/delete stuff
            destFile.copyFrom(this, Selectors.SELECT_SELF);
            if ((destFile.getType().hasContent()
                    && destFile.getFileSystem().hasCapability(Capability.SET_LAST_MODIFIED_FILE)
                    || destFile.getType().hasChildren()
                            && destFile.getFileSystem().hasCapability(Capability.SET_LAST_MODIFIED_FOLDER))
                    && fs.hasCapability(Capability.GET_LAST_MODIFIED)) {
            destFile.getContent().setLastModifiedTime(this.getContent().getLastModifiedTime());
        }
        deleteSelf();
    }

}
{% endcodeblock %}

这个方法也不复杂，移动文件有两种情况

1. 源文件和目标文件在同一个filesystem，使用`doRename`
2. 源文件和目标文件不在同一个filesystem，使用`copyFrom`

前面我们已经知道username，password，host，port相同的时候取到的就是同一个filesystem，所以这里判断源文件和目标文件是否在同一个filesystem也很简单，直接用==判断。
{% codeblock lang:java %}
@Override
public boolean canRenameTo(final FileObject newfile) {
    return fs == newfile.getFileSystem();
}
{% endcodeblock %}

`doRename`的实现原理就是调用`FTPClient的rename`方法，而这个`FTPClient`是通过`FTP协议`的`RNFR`和`RNTO`指令实现的。 `copyFrom`则是通过`FTP协议`中的`RETR`和`STOR`命令来下载上传实现的。



目前来看，文件移动都没什么问题，然而项目中导致移动文件慢的竟然是这个方法。
{% codeblock lang:java %}
@Override
public boolean exists() throws FileSystemException {
    return getType() != FileType.IMAGINARY;
}
{% endcodeblock %}
不管源文件与目标文件`是否在同一个文件系统`都会对源文件和目标文件执行这个`getType()`方法。这个方法最终会调用`FtpFileObject的doGetType()`方法。

{% codeblock lang:java %}
@Override
protected FileType doGetType() throws Exception {
    // VFS-210
    synchronized (getFileSystem()) {
        if (this.fileInfo == null) {
            getInfo(false);
        }
        if (this.fileInfo == UNKNOWN) {
            return FileType.IMAGINARY;
        } else if (this.fileInfo.isDirectory()) {
            return FileType.FOLDER;
        } else if (this.fileInfo.isFile()) {
            return FileType.FILE;
        } else if (this.fileInfo.isSymbolicLink()) {
            final FileObject linkDest = getLinkDestination();
            // VFS-437: We need to check if the symbolic link links back to the symbolic link itself
            if (this.isCircular(linkDest)) {
                // If the symbolic link links back to itself, treat it as an imaginary file to prevent following
                // this link. If the user tries to access the link as a file or directory, the user will end up with
                // a FileSystemException warning that the file cannot be accessed. This is to prevent the infinite
                // call back to doGetType() to prevent the StackOverFlow
                return FileType.IMAGINARY;
            }
            return linkDest.getType();
        }
    }
    throw new FileSystemException("vfs.provider.ftp/get-type.error", getName());
}
{% endcodeblock %}
上面已经说过，FtpFileObject的fileInfo没有初始化，所以这里会执行getInfo方法，而getInfo方法又会调用`getChildFile`
{% codeblock lang:java %}
private FTPFile getChildFile(final String name, final boolean flush) throws IOException {
    /*
    * If we should flush cached children, clear our children map unless we're in the middle of a refresh in which
    * case we've just recently refreshed our children. No need to do it again when our children are refresh()ed,
    * calling getChildFile() for themselves from within getInfo(). See getChildren().
    */
    if (flush && !inRefresh) {
        children = null;
    }
    // List the children of this file
    doGetChildren();
    // VFS-210
    if (children == null) {
        return null;
    }
    // Look for the requested child
    final FTPFile ftpFile = children.get(name);
    return ftpFile;
}
{% endcodeblock %}
就是这里，我们可以看到，获取某个文件时，会先获取`父路径的所有子文件`，然后从子文件中获取你要的那个文件。 如果你要的那个文件在一个文件非常多的目录里，而且`关闭了缓存`，你每获取这个目录的一个文件就要把目录里的所有文件列一次。
FTPClient是可以通过listFiles列出单个文件的，所以解决办法就是
1. 使用缓存
2. 不要用VFS了，直接用FTPClient的rename方法，直接起飞（仅限于同一个FTPClient，如果时跨文件服务器的需要FTPClient的上传下载实现）。
下面附上解决办法2的代码。
代码很简单，大多数都是解析URI的，全塞一个类里了，如果真要用建议把一些代码拆出来。
{% codeblock lang:java %}
import java.io.IOException;
import java.util.Arrays;
import java.util.Map;
import java.util.concurrent.atomic.AtomicReference;

import org.apache.commons.net.ftp.FTPClient;
import org.apache.commons.net.ftp.FTPFile;
import org.apache.commons.vfs2.FileSystemException;
import org.apache.commons.vfs2.FileSystemOptions;
import org.apache.commons.vfs2.provider.UriParser;
import org.apache.commons.vfs2.provider.ftp.FtpClientFactory;
import org.apache.commons.vfs2.provider.ftp.FtpFileSystemConfigBuilder;
import org.apache.commons.vfs2.util.Cryptor;
import org.apache.commons.vfs2.util.CryptorFactory;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.google.common.collect.Maps;

public class FtpUtil {
    private static Logger logger = LoggerFactory.getLogger(FtpUtil.class);
    
    private final static Map<Auth, AtomicReference<FTPClient>> clients = Maps.newConcurrentMap();
    
    public static boolean move(String src, String tar) throws IOException {
        FtpPath srcFtpPath = parse(src);
        FtpPath tarFtpPath = parse(tar);
        if (!srcFtpPath.auth.equals(tarFtpPath.auth)) {
            throw new UnsupportedOperationException("源目录和目标目录的ftp服务器连接信息不一致");
        }
        FTPClient ftpClient = getFTPClient(srcFtpPath.auth);
        try {
            return ftpClient.rename(srcFtpPath.path, tarFtpPath.path);
        } catch (IOException e) {
            closeConnection(ftpClient);
            throw e;
        } finally {
            putFTPClient(srcFtpPath.auth, ftpClient);
        }
    }
    
    public static FtpPath parse(String uri) throws FileSystemException {
        FtpPath ftpPath = new FtpPath();
        StringBuilder name = new StringBuilder();
        UriParser.extractScheme(uri, name);
        // Expecting "//"
        if (name.length() < 2 || name.charAt(0) != '/' || name.charAt(1) != '/') {
            throw new FileSystemException("vfs.provider/missing-double-slashes.error", uri);
        }
        name.delete(0, 2);
     // Extract userinfo, and split into username and password
        final String userInfo = extractUserInfo(name);
        final String userName;
        final String password;
        if (userInfo != null) {
            final int idx = userInfo.indexOf(':');
            if (idx == -1) {
                userName = userInfo;
                password = null;
            } else {
                userName = userInfo.substring(0, idx);
                password = userInfo.substring(idx + 1);
            }
        } else {
            userName = null;
            password = null;
        }
        
        String u = UriParser.decode(userName);
        String p = UriParser.decode(password);
    
        if (p != null && p.startsWith("{") && p.endsWith("}")) {
            try {
                final Cryptor cryptor = CryptorFactory.getCryptor();
                p = cryptor.decrypt(p.substring(1, p.length() - 1));
            } catch (final Exception ex) {
                throw new FileSystemException("Unable to decrypt password", ex);
            }
        }
        
        ftpPath.auth.username = u == null ? null : u.toCharArray();
        ftpPath.auth.password = p == null ? null : p.toCharArray();
        
        // Extract hostname, and normalise (lowercase)
        final String hostName = extractHostName(name);
        if (hostName == null) {
            throw new FileSystemException("vfs.provider/missing-hostname.error", uri);
        }
        ftpPath.auth.host = hostName.toLowerCase();
    
        // Extract port
        ftpPath.auth.port = extractPort(name, uri);
    
        // Expecting '/' or empty name
        if (name.length() > 0 && name.charAt(0) != '/') {
            throw new FileSystemException("vfs.provider/missing-hostname-path-sep.error", uri);
        }
        
        ftpPath.path = name.toString();
        return ftpPath;
    }
    
    /**
     * Extracts the user info from a URI.
     *
     * @param name string buffer with the "scheme://" part has been removed already. Will be modified.
     * @return the user information up to the '@' or null.
     */
    private static String extractUserInfo(final StringBuilder name) {
        final int maxlen = name.length();
        for (int pos = 0; pos < maxlen; pos++) {
            final char ch = name.charAt(pos);
            if (ch == '@') {
                // Found the end of the user info
                final String userInfo = name.substring(0, pos);
                name.delete(0, pos + 1);
                return userInfo;
            }
            if (ch == '/' || ch == '?') {
                // Not allowed in user info
                break;
            }
        }
    
        // Not found
        return null;
    }
    
    /**
     * Extracts the hostname from a URI.
     *
     * @param name string buffer with the "scheme://[userinfo@]" part has been removed already. Will be modified.
     * @return the host name or null.
     */
    private static String extractHostName(final StringBuilder name) {
        final int maxlen = name.length();
        int pos = 0;
        for (; pos < maxlen; pos++) {
            final char ch = name.charAt(pos);
            if (ch == '/' || ch == ';' || ch == '?' || ch == ':' || ch == '@' || ch == '&' || ch == '=' || ch == '+'
                    || ch == '$' || ch == ',') {
                break;
            }
        }
        if (pos == 0) {
            return null;
        }
    
        final String hostname = name.substring(0, pos);
        name.delete(0, pos);
        return hostname;
    }
    
    /**
     * Extracts the port from a URI.
     *
     * @param name string buffer with the "scheme://[userinfo@]hostname" part has been removed already. Will be
     *            modified.
     * @param uri full URI for error reporting.
     * @return The port, or -1 if the URI does not contain a port.
     * @throws FileSystemException if URI is malformed.
     * @throws NumberFormatException if port number cannot be parsed.
     */
    private static int extractPort(final StringBuilder name, final String uri) throws FileSystemException {
        if (name.length() < 1 || name.charAt(0) != ':') {
            return -1;
        }
    
        final int maxlen = name.length();
        int pos = 1;
        for (; pos < maxlen; pos++) {
            final char ch = name.charAt(pos);
            if (ch < '0' || ch > '9') {
                break;
            }
        }
    
        final String port = name.substring(1, pos);
        name.delete(0, pos);
        if (port.length() == 0) {
            throw new FileSystemException("vfs.provider/missing-port.error", uri);
        }
    
        return Integer.parseInt(port);
    }
    
    private static FTPClient getFTPClient(Auth key) throws IOException {
        AtomicReference<FTPClient> refClient = clients.getOrDefault(key, new AtomicReference<FTPClient>(null));
        
        FTPClient client = refClient.getAndSet(null);
        if (client == null || !client.isConnected()) {
            client = createClient(key);
        }
        return client;
    }
    
    private static FTPClient createClient(Auth key) throws IOException {
        FtpFileSystemConfigBuilder builder = FtpFileSystemConfigBuilder.getInstance();
        FileSystemOptions options = new FileSystemOptions();
        builder.setControlEncoding(options, "UTF-8");
        builder.setServerLanguageCode(options, "zh");
        builder.setPassiveMode(options, true);
        return FtpClientFactory.createConnection(key.host, key.port, key.username, key.password, null, options);
    }
    
    private static void putFTPClient(Auth key, FTPClient client) {
        AtomicReference<FTPClient> refClient = clients.getOrDefault(key, new AtomicReference<FTPClient>(null));
        
        if (!refClient.compareAndSet(null, client)) {
            closeConnection(client);
        }
    }
    
    private static void closeConnection(FTPClient client) {
        try {
            if (client.isConnected()) {
                client.disconnect();
            }
        } catch (final IOException e) {
            logger.error(e.getMessage(), e);
        }
    }
    
    private static class Auth {
        String host;
        int port;
        char[] username;
        char[] password;
        
        @Override
        public boolean equals(Object obj) {
            if (this == obj) {
                return true;
            }
            if (obj instanceof Auth) {
                Auth k = (Auth) obj;
                return this.host.equals(k.host) && this.port == k.port && Arrays.equals(this.username, k.username)
                        && Arrays.equals(this.password, k.password);
            }
            return false;
        }
        
        @Override
        public int hashCode() {
            int h = host.hashCode();
            h = 31 * h + port;
            h = 31 * h + username.hashCode();
            h = 31 * h + password.hashCode();
            return h;
        }
    }
    
    private static class FtpPath {
        Auth auth = new Auth();
        String path;
    }
}
{% endcodeblock %}