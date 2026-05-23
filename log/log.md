# tinyserver日志系统2026-5-21
## 功能
单例模式
## 私有成员
### 文件路径
char dir_name[128]; //日志文件所在目录
char log_name[128]; //日志文件基础名称
### 文件管理
FILE *m_fp;     //当前打开的日志文件指针
int m_split_lines;  //单个日志文件的最大行数
long long m_count;  //当前文件已写行数
int m_today;        //当前日志文件日起
### 缓冲区
int m_log_buf_size; //单条日志缓冲区大小
char *m_buf;    //动态分配的缓冲区
### 异步队列
block_queue<string> *m_log_queue;   //阻塞队列生产者消费者
bool m_is_async;    //是否开启异步模式
### 其他
locker m_mutex;     //互斥锁，保护文件写入
int m_close_log;    //日志是否开启
## 单例模式
    static Log *get_instance()
    {
        static Log instance;
        return &instance;
    }
c++11保证线程安全，采用懒汉，初次条用的时候创建，采用单例模式原因：
1. **资源唯一性**：日志文件只能有一个实例写入
2. **状态全局性**：配置、计数器、日期必须全局一致
3. **访问便利性**：任何模块都需要写日志，不想层层传递
4. **线程唯一性**：异步模式下只能有一个消费者线程
## 构造函数
Log::Log()
{
    m_count = 0;
    m_is_async = false;
}
初始化，处事写入行数为0,默认采用同步日志
##析构函数
Log::~Log()
{
    if(m_fp != nullptr)
    {
        fclose(m_fp);
    }
}
关闭打开的文件指针

## 初始化函数
bool Log::init(const char *file_name,int close_log,int log_buf_size,int split_lines,int max_queue_size)
### 异步模块初始化
    if(max_queue_size >= 1)
    {
        m_is_async = true;
        m_log_queue = new block_queue<string>(max_queue_size);
        pthread_t tid;
        pthread_create(&tid,nullptr,flush_log_thread,nullptr);
    }
如果异步队列长度超过1,则启用，设置异步标志，初始化异步队列，并创建对应线程。
### 缓冲区分配
    m_close_log = close_log;
    m_log_buf_size = log_buf_size;
    m_buf = new char[m_log_buf_size];
    memset(m_buf,'\0',m_log_buf_size);
    m_split_lines = split_lines;
根据参数启用或关闭日志，并分配缓冲区
### 时间处理
    time_t t = time(nullptr);
    struct tm *sys_tm = localtime(&t);
    struct tm my_tm = *sys_tm;
需要额外拷贝一份，因为后续调用tm指向的内容，该函数线程不安全
### 文件名构建逻辑
const char *p = strrchr(file_name,'/');
    char log_full_name[256] = {0};

    if(p == nullptr)
    {
        snprintf(log_full_name,255,"%d_%02d_%02d_%s",my_tm.tm_year + 1900,my_tm.tm_mon + 1,my_tm.tm_mday,file_name);        
    }
    else
    {
        strcpy(log_name,p + 1);
        strncpy(dir_name,file_name,p - file_name + 2);
        snprintf(log_full_name,255,"%s%d_%02d_%02d_%s",dir_name,my_tm.tm_year + 1900,my_tm.tm_mon + 1,my_tm.tm_mday,log_name);
    }
情况1：没有路径，只有文件名
直接在文件名前面添加年月日前缀
情况2：有路径，例如 "/var/log/server.log"
有路径：/var/log/2026_05_21_server.log
记录日期
    m_today = my_tm.tm_mday;
### 打开文件
    m_fp = fopen(log_full_name,"a");
    if(m_fp == nullptr)
    {
        return false;
    }
    return true;
## 写日志函数
void Log::write_log(int level,const char *format,...)
### 获取当前时间
struct timeval now = {0,0};
    gettimeofday(&now,nullptr);
    time_t t = now.tv_sec;
    struct tm *sys_tm = localtime(&t);
    struct tm my_tm = *sys_tm;
### 日志级别处理
    switch(level)
    {
    case 0:
        strcpy(s,"[debug]:");
        break;
    case 1:
        strcpy(s, "[info]:");
        break;
    case 2:
        strcpy(s, "[warn]:");
        break;
    case 3:
        strcpy(s, "[erro]:");
        break;
    default:
        strcpy(s, "[info]:");
        break;
    }
### 文件自动分割逻辑
m_mutex.lock();
    m_count++;

    if(m_today != my_tm.tm_mday || m_count % m_split_lines == 0)
    {
        char new_log[256] = {0};
        fflush(m_fp);
        fclose(m_fp);
        char tail[16] = {0};

        snprintf(tail,16,"%d_%02d_%02d_",my_tm.tm_year + 1900,my_tm.tm_mon + 1,my_tm.tm_mday);

        if (m_today != my_tm.tm_mday)
        {
            snprintf(new_log, 255, "%s%s%s", dir_name, tail, log_name);
            m_today = my_tm.tm_mday;
            m_count = 0;
        }
        else
        {
            snprintf(new_log, 255, "%s%s%s.%lld", dir_name, tail, log_name, m_count / m_split_lines);
        }
        m_fp = fopen(new_log, "a");
    }

    m_mutex.unlock();

对文件操作线上锁，然后根据有没有写满，日起是否超过关闭就的日志文件，构建新的日志文件。
### 日志格式化
    第一段：时间戳 + 日志级别

    第二段：用户传入的格式化内容

    最后手动添加换行符 \n
### 异步同步写入
m_mutex.unlock();

        if(m_is_async && !m_log_queue->full())
        {
            m_log_queue->push(log_str);
        }
        else
        {
            m_mutex.lock();
            fputs(log_str.c_str(),m_fp);
            m_mutex.unlock();
        }

        va_end(valst);
