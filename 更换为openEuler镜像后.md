# 确认 CANN 版本
cat /usr/local/Ascend/ascend-toolkit/latest/version.cfg | grep runtime_running
预期: 7.7.0.1.238:8.1.RC1
# 基础缓存+镜像源
cd /root/autodl-tmp
mkdir -p pip-cache huggingface conda_envs conda_pkg
# 写入当前用户 bashrc
echo 'export PIP_CACHE_DIR=/root/autodl-tmp/pip-cache'      >> ~/.bashrc
echo 'export HF_HOME=/root/autodl-tmp/autodl-fs/huggingface'         >> ~/.bashrc
echo 'export HF_ENDPOINT=https://hf-mirror.com'           >> ~/.bashrc
source ~/.bashrc

# conda 缓存迁移
cat > ~/.condarc <<EOF
envs_dirs:
  - /root/autodl-tmp/conda_envs
pkgs_dirs:
  - /root/autodl-tmp/conda_pkg
EOF
# 创建并激活conda
conda create -n rag python=3.10 -y
conda init bash
source ~/.bashrc
conda activate rag
# 安装MindIE 1.0.0（CANN 8.1.RC1 专用包）
cd /root/autodl-tmp
pip install torch==2.6.0
pip install pyyaml
pip install setuptools
pip install torch-npu==2.6.0
# 查看MindIE LLM 是否存在
find / -name "mindie_llm*.whl" 2>/dev/null
pip install /usr/local/Ascend/mindie/2.0.RC2/mindie-llm/bin/mindie_llm-2.0rc2-py3-none-any.whl
# 首先验证mindie_llm导入
python - <<'PY'
import mindie_llm
print(dir(mindie_llm))
PY

# 验证
python test/test_torchnpu.py
## 应当安装
pip install tokenizers==0.19.1
pip install "transformers<4.41" --force-reinstall
pip install --force-reinstall "numpy<2.0"
# 清理CANN缓存
rm -rf ~/.cache/ascend-graph/* ~/.cache/atc/*
# 设置CANN环境变量
使用正确的setenv.bash文件
source /usr/local/Ascend/ascend-toolkit/latest/aarch64-linux/bin/setenv.bash
# 需要安装数据库
包管理器是：dnf
dnf update -y
# 安装编译工具及相关依赖
sudo dnf install -y perl-ExtUtils-Embed readline-devel python3-devel pam-devel libxml2-devel libxslt-devel openldap-devel lz4-devel llvm-devel systemd-devel container-selinux selinux-policy-devel openssl-devel clang-devel flex bison glibc-devel gcc-c++ gcc cmake lsof net-tools libicu-devel tar
# 下载并解压 PostgreSQL 源码
wget https://ftp.postgresql.org/pub/source/v16.3/postgresql-16.3.tar.gz
tar -zxvf postgresql-16.3.tar.gz
cd postgresql-16.3
# 编译和安装 PostgreSQL
./configure --prefix=/home/pgdata
make -j$(nproc) world && make install-world
# 将路径加到PATH
echo 'export PATH=/home/pgdata/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
# 创建数据目录并初始化
which psql
mkdir -p /root/autodl-tmp/pgdata/data
chown -R postgres:postgres /root/autodl-tmp/pgdata
chmod 700 /root/autodl-tmp/pgdata/data
# 因为之前安装过15.13版本的PostgreSQL，所以现在还是启动了15.13版本的数据库
# 开放root的权限
chmod 755 /root  
# 切用户
su - postgres
# 初始化
initdb -D /root/autodl-tmp/pgdata/data -U postgres
# 启动数据库
pg_ctl -D /root/autodl-tmp/pgdata/data -l logfile start
# 或
su - postgres -c "psql -U postgres"
# 查看目前的postgresSQL版本,发现还在15.14版本下面，现在切换到16.3
psql -c "SHOW server_version;"
查看数据目录位置:
su - postgres -c "psql -c 'SHOW data_directory;'"
su - postgres -c "/usr/bin/pg_ctl -D /root/autodl-tmp/pgdata/data stop -m fast"
ps aux | grep 'postgres.*15'
rm -rf /root/autodl-tmp/pgdata/data/*
# 重新初始化16.3版本数据库
su - postgres
/home/pgdata/bin/initdb -D /root/autodl-tmp/pgdata/data -U postgres \
                        --encoding=UTF8 --locale=C
启动16.3：
/home/pgdata/bin/pg_ctl -D /root/autodl-tmp/pgdata/data \
                        -l /root/autodl-tmp/pgdata/logfile start
# 查看目前的版本：
su - postgres -c "psql -h localhost -c 'SHOW server_version;'"
## 目前存在15.14版本的数据库和16.3版本的数据库：
### 想调用 15
/usr/bin/psql -c "SHOW server_version;"
### 想调用 16
/home/pgdata/bin/psql -c "SHOW server_version;"
# 针对 16.3 重新编译 / 安装扩展文件
yum install -y gcc make
用 16 的 pg_config 编译并安装：
cd /root/autodl-tmp/pgvector
# 清理旧对象（如果有）
make clean

# 指定 16 的 pg_config 路径
make PG_CONFIG=/home/pgdata/bin/pg_config
make PG_CONFIG=/home/pgdata/bin/pg_config install
# 进库启用扩展
su - postgres
#psql -h localhost -U postgres
客户端和服务器都使用16.3版本：
/home/pgdata/bin/psql -h localhost -U postgres
## 然后运行：
    CREATE EXTENSION vector;
    SELECT extversion FROM pg_extension WHERE extname = 'vector';
## 验证功能，验证向量功能是否正常
-- 创建一个测试表
    CREATE TABLE items (id serial PRIMARY KEY, embedding vector(3));

-- 插入一条 3 维向量
    INSERT INTO items (embedding) VALUES ('[1,2,3]');

-- 查询最邻近向量（L2 距离）
    SELECT * FROM items ORDER BY embedding <-> '[1,1,1]' LIMIT 1;

## 退出
    \q

# 使用mindie推理大模型：
# 1. 加载 CANN 环境
source /usr/local/Ascend/ascend-toolkit/set_env.sh

