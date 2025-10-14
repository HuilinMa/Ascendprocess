# 官网文档参考: https://ascend.github.io/docs/sources/ascend/quick_install.html
本服务器NPU型号确定为：Atlas 300T A2
# 确认 CANN 版本
uname -m
cat /usr/local/Ascend/ascend-toolkit/latest/version.cfg | grep runtime_running
预期: 7.7.0.1.238:8.1.RC1
# 基础缓存+镜像源
cd /root/autodl-tmp
mkdir -p pip-cache autodl-fs conda_envs conda_pkg
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

# 放进动态链接器的搜索路径：
临时生效：
export LD_LIBRARY_PATH=/home/pgdata/lib:$LD_LIBRARY_PATH
永久生效：
export LD_LIBRARY_PATH=/home/pgdata/lib:$LD_LIBRARY_PATH
source ~/.bashrc

# 测试PostgreSQL以及pgvector
# 切用户
su - postgres
启动16.3版本：
/home/pgdata/bin/pg_ctl -D /root/autodl-tmp/pgdata/data \
                        -l /root/autodl-tmp/pgdata/logfile start
# 查看目前的版本：
su - postgres -c "psql -h localhost -c 'SHOW server_version;'"
已经切过用户：psql -h localhost -c 'SHOW server_version;'
# 查看运行状态：
/home/pgdata/bin/psql -p 5432 -U postgres -c "SELECT version();"
## 然后运行：
    使用16.3版本：
    /home/pgdata/bin/psql -h localhost -U postgres
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
# 创建环境：
conda create -n uchat python=3.10 -y
conda activate uchat
pip install -r requirements.txt
# 安装算子包
wget "https://ascend-repo.obs.cn-east-2.myhuaweicloud.com/CANN/CANN 8.1.RC1/Ascend-cann-kernels-910b_8.1.RC1_linux-aarch64.run"
sh Ascend-cann-kernels-910b_8.1.RC1_linux-aarch64.run --install
# 安装依赖：
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple attrs cython numpy==1.24.0 decorator sympy cffi pyyaml pathlib2 psutil protobuf==5.27.2 scipy requests absl-py
pip install torch==2.2.0 torchvision==0.17.0 --index-url https://download.pytorch.org/whl/cpu
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple torch-npu==2.2.0
pip install attrs decorator tornado
pip install "transformers>=4.33.0,<4.51.0"
# 设置CANN环境变量
使用正确的setenv.bash文件
echo source /usr/local/Ascend/ascend-toolkit/latest/aarch64-linux/bin/setenv.bash    >>~/.bashrc
echo source /usr/local/Ascend/ascend-toolkit/set_env.sh >>~/.bashrc
source ~/.bashrc
# 测试torch\torch-npu:
python /root/autodl-tmp/test/test_torchnpu.py
预期输出：
torch      : 2.x.0+cpu
torch_npu  : 2.x.0
NPU available: True
NPU device count: 1
Current NPU device: 0

# 安装Imdeploy
# 说明：在「LMDeploy 新版本要求」与「昇腾生态上限」之间，选择交集：2.2.0版本的torch/torch-npu + v0.9.2 Imdeploy。
cd /root/autodl-tmp
git clone https://github.com/InternLM/lmdeploy.git
cd lmdeploy
git checkout v0.9.2                # 标签存在，但无 wheel
LMDEPLOY_TARGET_DEVICE=ascend pip install -v --no-build-isolation -e .
## 测试：
python -c "import lmdeploy; print(lmdeploy.__version__)"

# 测试中出现的问题:
环境混用：
echo unset PYTHONPATH   >>~/.bashrc
echo unset PYTHONHOME   >>~/.bashrc
source ~/.bashrc
# 测试
python /root/autodl-tmp/test/test_LMDeploy.py

## 系统默认路径里并没有 libpq.so.5,把 /home/pgdata/lib 写进系统配置:
echo "/home/pgdata/lib" > /etc/ld.so.conf.d/pgdata.conf
ldconfig

# 运行Uchat
# 启动数据库，需要手动启动数据库：
su - postgres
启动16.3版本：
/home/pgdata/bin/pg_ctl -D /root/autodl-tmp/pgdata/data \
                        -l /root/autodl-tmp/pgdata/logfile start
查看数据库是否启动：
pgrep -af postgres
查看端口号:
grep -nE 'listen_addresses|port' /root/autodl-tmp/pgdata/data/postgresql.conf
# 出现 Out of NPU memory 或者想预留更大余地 → 开启：
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
# 后台使用Lmdeploy启动7B的通义模型（关掉图编译，用 eager 模式即可）：
lmdeploy serve api_server \
        /root/autodl-tmp/autodl-fs/qwen-7b-chat \
        --backend pytorch \
        --device ascend \
        --tp 1 \
        --server-name 0.0.0.0 \
        --server-port 8000 \
        --eager-mode \
        --log-level ERROR
# 测试
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "/root/autodl-tmp/autodl-fs/qwen-7b-chat",
    "messages": [{"role": "user", "content": "hi"}],
    "max_tokens": 32
  }'

# 启动程序
cd uchat
python chat.py

# 启动前端界面
streamlit run web/entrance.py 

# 知乎参考文档（非必须）：
https://zhuanlan.zhihu.com/p/25147199560

# 还未安装libreoffice！


