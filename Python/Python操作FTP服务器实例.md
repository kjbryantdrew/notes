```python
# ftp登陆连接

from ftplib import FTP # 加载ftp模块
ftp=FTP() # 设置变量
ftp.set_debuglevel(2) # 打开调试级别2，显示详细信息
ftp.connect(“IP”,”port”) # 连接的ftp sever和端口
ftp.login(“user”,”password”) # 连接的用户名，密码
print ftp.getwelcome() # 打印出欢迎信息
ftp.cmd(“xxx/xxx”) # 进入远程目录
bufsize=1024 # 设置的缓冲区大小
filename=”filename.txt” # 需要下载的文件
file_handle=open(filename,”wb”).write #以写模式在本地打开文件
ftp.retrbinaly(“RETR filename.txt”,file_handle,bufsize) # 接收服务器上文件并写入本地文件
ftp.set_debuglevel(0) # 关闭调试模式
ftp.quit() # 退出ftp
 
ftp相关命令操作
ftp.cwd(pathname) # 设置FTP当前操作的路径
ftp.dir() # 显示目录下所有目录信息
ftp.nlst() # 获取目录下的文件
ftp.mkd(pathname) # 新建远程目录
ftp.pwd() # 返回当前所在位置
ftp.rmd(dirname) # 删除远程目录
ftp.delete(filename) # 删除远程文件
ftp.rename(fromname, toname) # 将fromname修改名称为toname。
ftp.storbinaly(“STOR filename.txt”,file_handel,bufsize) # 上传目标文件
ftp.retrbinary(“RETR filename.txt”,file_handel,bufsize) # 下载FTP文件
```

示例代码：
```python
#ftp演示，首先要在本机或远程服务器开启ftp功能
import sys,os,ftplib,socket,hashlib
print("=====================FTP客户端=====================");
HOST = ''  # FTP主机地址
username = ''  # 用户名
password = ''  # 密码
buffer_size = 8192  # 缓冲区大小
 
# 连接登陆
def connect():
    try:
        ftp = ftplib.FTP(HOST)  # 实例化FTP对象
        ftp.login(username, password)  # 登录
        ftp.set_pasv(False)  # 如果被动模式由于某种原因失败，请尝试使用活动模式。
        print(ftp.getwelcome())
        print('已连接到： %s' % HOST)
        return ftp
    except (socket.error,socket.gaierror):
        print("FTP登陆失败，请检查主机号、用户名、密码是否正确")
        sys.exit(0)
 
# 获取目录下文件或文件夹详细信息
def dirInfo(ftp):
    ftp.dir()
 
# 获取目录下文件或文件夹的列表信息，并清洗去除“. ..”
def nlstListInfo(ftp):
    files_list = ftp.nlst()
    return [file for file in files_list if file != "." and file !=".."]
 
# 获取当前路径
def pwdinfo(ftp):
    pwd_path = ftp.pwd()
    print("FTP当前路径:", pwd_path)
    return ftp.pwd()
 
# 中断并退出
def disconnect(ftp):
    ftp.quit()  # FTP.close()：单方面的关闭掉连接。FTP.quit():发送QUIT命令给服务器并关闭掉连接
 
# 目录跳转
def pwdSkip(ftp,dirPathName):
    """
    跳转到指定目录
    :param ftp: 调用connect()方法的变量
    :param dirPathName: FTP服务器的绝对路径
    :return:
    """
    try:
        ftp.cwd(dirPathName)             # 重定向到指定路径
    except ftplib.error_perm:
        print('不可以进入目录："%s"' % dirPathName)
    print("当前所在位置:%s" % ftp.pwd())  # 返回当前所在位置
 
# 判断文件与目录
def checkFileDir(ftp,file_name):
    """
    判断当前目录下的文件与文件夹
    :param ftp: 实例化的FTP对象
    :param file_name: 文件名/文件夹名
    :return:返回字符串“File”为文件，“Dir”问文件夹，“Unknow”为无法识别
    """
    rec = ""
    try:
        rec = ftp.cwd(file_name)   # 需要判断的元素
        ftp.cwd("..")   # 如果能通过路劲打开必为文件夹，在此返回上一级
    except ftplib.error_perm as fe:
        rec = fe # 不能通过路劲打开必为文件，抓取其错误信息
    finally:
        if "Not a directory" in str(rec):
            return "File"
        elif "Current directory is" in str(rec):
            return "Dir"
        else:
            return "Unknow"
 
# 文件名加密
def fileNameMD5(filepath):
    """
    文件名加密MD5
    :param filepath: 本地需要加密的文件绝对路径
    :return: 返回加密后的文件名
    """
    file_name = os.path.split(filepath)[-1]     # 获取文件全名
    encryption = hashlib.md5()      # 实例化MD5
    try:
        file_format = os.path.splitext(file_name)[-1]  # 截取文件格式
        file_name_section = os.path.splitext(file_name)[0]
    except IndexError as e:
        print('%s 文件有误！未发现后缀格式！报错信息：%s' %(file_name,e))
        return "无后缀格式的文件"
    else:
        encryption.update(file_name_section.encode('utf-8'))
        newname = "zzuliacgn_" + encryption.hexdigest() + "." + file_format
        print('MD5加密前为 ：' + file_name)
        print('MD5加密后为 ：' + newname)
        return newname
 
# 上传文件
def upload(ftp, filepath,file_name = None):
    """
    上传文件
    :param ftp: 实例化的FTP对象
    :param filepath: 上传文件的本地路径
    :param file_name: 上传后的文件名（结合fileNameMD5()方法用）
    :return:
    """
    f = open(filepath, "rb")
    if file_name == None:
        file_name = os.path.split(filepath)[-1]
 
    if find(ftp, file_name) or file_name == "无后缀格式的文件":
        print("%s 已存在或识别为无后缀格式的文件,上传终止"%file_name) # 上传本地文件,同名文件会替换
        return False
    else:
        try:
            ftp.storbinary('STOR %s'%file_name, f, buffer_size)
            print('成功上传文件： "%s"' %file_name)
        except ftplib.error_perm:
            return False
    return True
 
# 下载文件
def download(ftp, filename):
    """
    下载文件
    :param ftp: 实例化的FTP对象
    :param filename: 需要从FTP下载的文件名
    :return:
    """
    f = open(filename,"wb").write
    try:
        ftp.retrbinary("RETR %s"%filename, f, buffer_size)
        print('成功下载文件： "%s"' % filename)
    except ftplib.error_perm:
        return False
    return True
 
# 查找是否存在指定文件或目录
def find(ftp,filename):
    """
    查找是否存在指定文件或目录
    :param ftp:  实例化的FTP对象
    :param filename:  需要查询是否存在的文件名
    :return:
    """
    ftp_f_list = ftp.nlst()  # 获取目录下文件、文件夹列表
    if filename in ftp_f_list:
        return True
    else:
        return False
 
# 检查是否有存在指定目录并创建
def mkdir(ftp,dirpath):
    """
    检查是否有存在指定目录并创建（懒人用的方法）
    :param ftp: 实例化的FTP对象
    :param dirpath: 需要创建的文件夹名及路径
    :return:
    """
    if find(ftp, dirpath):
        print("%s目录已存在！自行跳转到该目录！"%dirpath)
        pwdSkip(ftp, dirpath)       # 设置FTP当前操作的路径
    else:
        print("未发现%s同名文件夹！" % dirpath)
        try:
            ftp.mkd(dirpath)    # 新建远程目录
            print("创建新目录%s！并自行跳转到该目录！" % dirpath)
            pwdSkip(ftp, dirpath)    # 设置FTP当前操作的路径
        except ftplib.error_perm:
            print("目录已经存在或无法创建")
 
# 删除目录下文件
def DeleteFile(ftp,filepath = "/",file_name = None):
    """
    删除目录下文件,字面意思
    :param ftp: 实例化的FTP对象
    :param filepath: 操作的路径，默认值“/”
    :param file_name: 需要删除的文件名，若不填，默认删除目录下所有的文件
    :return:
    """
    pwdSkip(ftp,filepath)   # 跳转到操作目录
    if find(ftp,file_name) and file_name != None:
        ftp.delete(file_name)   # 删除文件
        print("%s 已删除！"%file_name)
    elif file_name == None:
        # print("file_name:%s 将删除 %s 目录下所有文件（目录除外）！" % (file_name,ftp.pwd()))
        filelist = nlstListInfo(ftp)
        for i in filelist:
            if checkFileDir(ftp,i) == "File":
                ftp.delete(i)  # 删除文件
                print("%s 是文件，已删除！" % i)
            elif checkFileDir(ftp,i) == "Dir":
                print("%s 是文件夹" % i)
            else:
                print("%s 无法识别，跳过" % i)
    else:
        print("%s 未找到，删除中止！" % file_name)
 
# 删除目录下的空文件夹
def DeleteDir(ftp,dirpath,dir_name = None):
    """
    删除目录下的空文件夹，非空文件夹和文件不会删除
    :param ftp: 实例化的FTP对象
    :param dirpath: 操作的路径
    :param dir_name: 需要删除的文件夹名，若不填，默认删除目录下所有的文件
    :return:
    """
    pwdSkip(ftp, dirpath)  # 跳转到操作目录
    if find(ftp,dir_name) and dir_name != None:
        ftp.delete(dir_name)   # 删除文件
        print("%s 已删除！"%dir_name)
    elif dir_name == None:
        # print("file_name:%s 将删除 %s 目录下所有文件夹（文件除外）！" % (dir_name,ftp.pwd()))
        filelist = nlstListInfo(ftp)
        for i in filelist:
            if checkFileDir(ftp,i) == "File":
                print("%s 是文件，不与理会！" % i)
            elif checkFileDir(ftp,i) == "Dir":
                try:
                    ftp.rmd(i)  # 删除文件)  # 重定向到指定路径
                    print("%s 是文件夹，已删除！" % i)
                except ftplib.error_perm as e:
                    print('无法删除 %s，文件夹里似乎还有东西！报错信息："%s"' %(i,e))
            else:
                print("%s 无法识别，跳过" % i)
    else:
        print("%s 未找到，删除中止！" % dir_name)
 
# 查询目录下的非空文件夹
def listdir(ftp,fulllist):
    dir_list = []
    haveDir_list = []
    for file in fulllist:  # 遍历文件夹
        if checkFileDir(ftp, file) == "Dir":  # 判断是否是文件夹，是文件夹才打开
            dir_list.append(file)
    for file in dir_list:  # 遍历文件夹
        ftp.cwd(file)
        if nlstListInfo(ftp) != []:  # 判断是否是文件夹，是文件夹才打开
            haveDir_list.append(file)
        ftp.cwd("..")
    return haveDir_list
 
# 递归器
def TraversingIter(ftp,path = "/"):
    ftp.cwd(path)
    haveDir_list = listdir(ftp, nlstListInfo(ftp))
    if haveDir_list == []:
        print("%s"%(ftp.pwd()))
        return ftp.pwd()
    it = iter(haveDir_list)
    return TraversingIter(ftp,next(it))
 
# 遍历删除文件夹或文件
def DeleteDirFiles(ftp,dirpath = "/"):
    """
    搜索（遍历）删除文件夹或文件
    :param ftp: 调用connect()方法的变量
    :param dirpath:限定搜索的路径范围，默认值为“/”根目录
    :return:
    """
    path = ""
    pwdSkip(ftp, dirpath)
    while dirpath != path:
        path = TraversingIter(ftp, dirpath)
        DeleteFile(ftp, path)    #删除该目录下所有文件
        DeleteDir(ftp, path)    #删除该目录下所有文件
    print("删除完毕!%s目录下已清空"%dirpath)
 
 
# 主函数
def main():
    # 连接登陆ftp（必须的）
    ftp = connect()
    try:
        # 以下是各种方法的使用示例
 
        # 获取目录下文件或文件夹详细信息
        dirInfo(ftp)
 
        # 获取目录下文件或文件夹的列表信息，并清洗去除“. ..”
        nlstListInfo(ftp)
 
        # 获取当前路径
        pwdinfo(ftp)
 
        # 目录跳转
        pwdSkip(ftp, "/")
 
        # 判断文件与目录
        file_name = "test"  # 需要判断的文件名或者是文件夹名
        checkFileDir(ftp, file_name)
 
        # 文件名加密并上传
        filepath = "D:\\workspace\\PythonSpace\\Spyder\\RequestsSpyder\\1.jpg"  # 需要加密的文件名的绝对路径
        file_name = fileNameMD5(filepath)   # 此函数，仅用于本地，一般结合上传文件方法使用
        upload(ftp, filepath, file_name)    # 上文件
 
        # 下载文件
        dirPathName = "/"
        pwdSkip(ftp, dirPathName)   # 跳转到FTP上的/目录，根据自己的需要修改
        filename = "test"   # 需要从FTP上下载的文件名
        download(ftp, filename)
 
        # 查找是否存在指定文件或目录
        filename = "test"
        if find(ftp, filename):     # 此函数，一般结合其他方法使用,返回值为True&False
            print("存在")
        else:
            print("不存在")
 
        # 检查是否有存在指定目录并创建
        dirpath = "test"        # 想要创建的文件夹名
        mkdir(ftp, dirpath)     # 在FTP新建一个名为“test”的文件夹
 
        # 删除目录下文件
        filepath = "/test"      # 需要删除的文件所在的路径
        file_name = "test233.jpg"       # 需要删除的文件名
        DeleteFile(ftp, filepath, file_name)    # 如果filepath，file_name不传值，默认filepath为“/”目录，file_name默认为None(详见该方法的说明)
 
        # 删除目录下的文件夹
        filepath = "/test"
        file_name = "test233.jpg"
        DeleteFile(ftp, filepath, file_name)        # 用法同上，只是它只删除空的文件夹，不删除文件和非空文件夹
 
        # 查询目录下的非空文件夹
        listdir(ftp, nlstListInfo(ftp))     # 此方法一般不单独使用，需结合 nlstListInfo()方法
 
        # 遍历删除文件夹或文件
        DeleteDirFiles(ftp)        # 删光光！
        # ....
    except ftplib.error_perm as e:
        print(e)
    finally:
        # 退出ftp（必须的）
        disconnect(ftp)
 
 
if __name__ == '__main__':
    main()
```

参考：[Python：使用操作FTP服务器实例](https://desirefire.github.io/2018/10/26/Python%EF%BC%9A%E4%BD%BF%E7%94%A8%E6%93%8D%E4%BD%9CFTP%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%AE%9E%E4%BE%8B%EF%BC%81%EF%BC%88%E4%B8%8A%E4%BC%A0%EF%BC%8C%E4%B8%8B%E8%BD%BD%EF%BC%8C%E9%81%8D%E5%8E%86%E5%88%A0%E9%99%A4%E7%AD%89%EF%BC%89/)
