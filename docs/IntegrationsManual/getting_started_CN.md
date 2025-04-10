# 集成 Visionatrix：快速入门

本指南将帮助您开始通过编程方式与 Visionatrix 交互。

!!! note

    **SDXL Lighting** 和 **Remove Background (Birefnet)** 流程应安装在 Visionatrix 实例上。

    本指南仅涵盖如何从流程创建任务、获取其进度并接收结果。

!!! info

    我们不断努力使我们的 API 更易于使用，代码由 Claude、Qwen 或 ChatGPT 自动生成。

    然而，本指南主要关注任务的生命周期，并不描述如何为指定的流程自动创建 Gradio 应用程序或其他类似主题。

## 目录

1. [身份验证](#authentication)
2. [任务生命周期](#task-lifecycle)
3. [完整的 Python 示例](#full-working-examples-in-python)
    - [示例 1：使用 "SDXL Lighting" 流程](#example-1-using-the-sdxl-lighting-flow)
    - [示例 2：使用 "Remove Background (Birefnet)" 流程](#example-2-using-the-remove-background-birefnet-flow)

## 身份验证

Visionatrix *目前* 仅支持 **Basic Authentication**。根据 Visionatrix 的运行模式，您可能需要或不需要提供身份验证凭据：

- **SERVER 模式**：如果 Visionatrix 在 SERVER 模式下运行，则必须在 API 请求中指定用户名和密码。
- **DEFAULT 模式**：如果 Visionatrix 在 DEFAULT 模式下运行，则无需身份验证。

为了本指南的目的，我们假设需要身份验证。

默认情况下，在开发环境中测试 Visionatrix 时，我们在 **SERVER 模式** 下使用 `admin` 作为用户名和密码。如果您的实际凭据不同，请替换它们。

## 任务生命周期

Visionatrix 中任务的典型生命周期包括以下步骤：

1. **创建任务**：您发送一个 `PUT` 请求到 `/vapi/tasks/create/{name}` 端点，其中 `{name}` 是要使用的流程 ID。请求必须使用 `multipart/form-data` 并包含流程所需的参数。

   - 对于 **SDXL Lighting** (`sdxl_lighting`) 流程，所需参数为：
     - `prompt` (字符串)：用于图像生成的文本提示。
     - `steps_number` (字符串)：要使用的步骤数。
   - **可选参数**：
     - `negative_prompt` (字符串)：用于图像生成的“负面”文本提示。
     - `seed` (整数)：随机数生成的种子。如果不指定，则使用随机种子。

   - 对于 **Remove Background (Birefnet)** (`remove_background_birefnet`) 流程，所需参数为：
     - `input_image` (文件)：要从中删除背景的图像文件。

   - **其他参数**：
     - 还有其他可选参数，如 `webhook_url`、`webhook_headers`、`translate`、`group_scope` 等。这些参数不在初学者指南中覆盖。

2. **检查任务进度**：创建任务后，可以使用 `/vapi/tasks/progress/{task_id}` 端点检查其进度。响应包括任务的 `progress`、`error`（如果有）和 `outputs` 列表。

   - **进度值**：
     - `0.0`：任务已排队且尚未开始。
     - 介于 `0.1` 和 `99.9`：任务正在进行中。
     - `100.0`：任务已完成。

   - **错误处理**：
     - 如果 `error` 字段不为空，则任务遇到错误。
     - 您可以根据错误消息重试任务或调查问题。

3. **检索任务结果**：一旦任务完成（`progress` 达到 `100.0`），可以使用 `/vapi/tasks/results` 端点检索结果。任务详细信息中的 `outputs` 包含输出节点列表，每个节点都有一个 `comfy_node_id`。您应该遍历所有输出以检索所有结果。

   - 要检索每个结果，您发送一个 `GET` 请求到 `/vapi/tasks/results`，附带 `task_id` 和 `node_id`（即来自输出的 `comfy_node_id`）。
   - 结果文件可以保存到本地。结果文件的格式取决于流程和输出节点的类型。

!!! note

    我们目前正在为流程自动生成 OpenAPI 规范：[Flows API](https://visionatrix.github.io/VixFlowsDocs/swagger.html?urls.primaryName=Visionatrix+Flows+API)。

    您可以在那里轻松查看流程参数。

## 完整的 Python 示例

以下是使用 `httpx` 库的完整 Python 示例。这些脚本演示了如何创建任务、检查其进度并检索所有结果，分别针对 `sdxl_lighting` 和 `remove_background_birefnet` 流程。

在运行脚本之前，请确保已安装 `httpx` 库：

```bash
pip install httpx
```

### 示例 1：使用 "SDXL Lighting" 流程

```python
import httpx
import time

# 替换为您的 Visionatrix 服务器 URL
base_url = "http://localhost:8288"
username = "admin"
password = "admin"


def create_sdxl_lighting_task():
    # 任务参数
    params = {
        'prompt': '美丽的夕阳映照在群山上',
        'steps_number': '8 steps',
        # 'negative_prompt' 是可选的
        # 'negative_prompt': '模糊, 低分辨率',
        # 'seed' 是可选的；如果不指定，则使用随机种子
        # 'seed': '12345',
    }

    # 将参数转换为 `files` 参数期望的格式
    files = {key: (None, value) for key, value in params.items()}

    # 创建任务
    response = httpx.put(
        f"{base_url}/vapi/tasks/create/sdxl_lighting",
        auth=(username, password),
        files=files
    )

    # 检查请求是否成功
    if response.status_code == 200:
        data = response.json()
        task_ids = data.get('tasks_ids', [])
        if task_ids:
            task_id = task_ids[0]
            print("任务创建成功。任务 ID:", task_id)
            return task_id
        else:
            print("未返回任务 ID。")
    else:
        print("任务创建失败:", response.text)
    return None


def check_task_progress(task_id):
    while True:
        response = httpx.get(
            f"{base_url}/vapi/tasks/progress/{task_id}",
            auth=(username, password)
        )

        if response.status_code == 200:
            task_details = response.json()
            progress = task_details.get('progress', 0)
            error = task_details.get('error', '')
            print(f"任务 {task_id} 进度: {progress}%")
            if error:
                print(f"任务 {task_id} 遇到错误: {error}")
                return None
            if progress >= 100:
                print("任务完成。")
                outputs = task_details.get('outputs', [])
                return outputs  # 返回 outputs 以避免额外的服务器调用
        else:
            print("获取任务进度失败:", response.text)
            return None

        # 等待后再轮询
        time.sleep(5)


def retrieve_task_results(task_id, outputs):
    if outputs:
        for output in outputs:
            node_id = output.get('comfy_node_id')
            if node_id is not None:
                # 检索每个输出节点的结果
                params = {
                    'task_id': task_id,
                    'node_id': node_id
                }
                result_response = httpx.get(
                    f"{base_url}/vapi/tasks/results",
                    auth=(username, password),
                    params=params
                )
                if result_response.status_code == 200:
                    # 将结果保存到文件
                    result_filename = f"result_{task_id}_{node_id}.png"
                    with open(result_filename, 'wb') as f:
                        f.write(result_response.content)
                    print(f"结果保存到 {result_filename}")
                else:
                    print(f"检索节点 {node_id} 的结果失败:", result_response.text)
            else:
                print("未找到输出节点 ID。")
    else:
        print("任务详细信息中未找到输出。")


def main():
    task_id = create_sdxl_lighting_task()
    if task_id:
        outputs = check_task_progress(task_id)
        if outputs is not None:
            retrieve_task_results(task_id, outputs)


if __name__ == "__main__":
    main()
```

### 示例 2：使用 "Remove Background (Birefnet)" 流程

```python
import httpx
import time

# 替换为您的 Visionatrix 服务器 URL
base_url = "http://localhost:8288"
username = "admin"
password = "admin"
input_image_path = "image.jpg"


def create_remove_background_task():
    # 以二进制模式打开图像文件
    with open(input_image_path, 'rb') as image_file:
        files = {
            'input_image': image_file
        }

        # 创建任务
        response = httpx.put(
            f"{base_url}/vapi/tasks/create/remove_background_birefnet",
            auth=(username, password),
            files=files
        )

    # 检查请求是否成功
    if response.status_code == 200:
        data = response.json()
        task_ids = data.get('tasks_ids', [])
        if task_ids:
            task_id = task_ids[0]
            print("任务创建成功。任务 ID:", task_id)
            return task_id
        else:
            print("未返回任务 ID。")
    else:
        print("任务创建失败:", response.text)
    return None


def check_task_progress(task_id):
    while True:
        response = httpx.get(
            f"{base_url}/vapi/tasks/progress/{task_id}",
            auth=(username, password)
        )

        if response.status_code == 200:
            task_details = response.json()
            progress = task_details.get('progress', 0)
            error = task_details.get('error', '')
            print(f"任务 {task_id} 进度: {progress}%")
            if error:
                print(f"任务 {task_id} 遇到错误: {error}")
                return None
            if progress >= 100:
                print("任务完成。")
                outputs = task_details.get('outputs', [])
                return outputs  # 返回 outputs 以避免额外的服务器调用
        else:
            print("获取任务进度失败:", response.text)
            return None

        # 等待后再轮询
        time.sleep(5)


def retrieve_task_results(task_id, outputs):
    if outputs:
        for output in outputs:
            node_id = output.get('comfy_node_id')
            if node_id is not None:
                # 检索每个输出节点的结果
                params = {
                    'task_id': task_id,
                    'node_id': node_id
                }
                result_response = httpx.get(
                    f"{base_url}/vapi/tasks/results",
                    auth=(username, password),
                    params=params
                )
                if result_response.status_code == 200:
                    # 将结果保存到文件
                    result_filename = f"result_{task_id}_{node_id}.png"
                    with open(result_filename, 'wb') as f:
                        f.write(result_response.content)
                    print(f"结果保存到 {result_filename}")
                else:
                    print(f"检索节点 {node_id} 的结果失败:", result_response.text)
            else:
                print("未找到输出节点 ID。")
    else:
        print("任务详细信息中未找到输出。")


def main():
    task_id = create_remove_background_task()
    if task_id:
        outputs = check_task_progress(task_id)
        if outputs is not None:
            retrieve_task_results(task_id, outputs)


if __name__ == "__main__":
    main()
```

---

**注意事项**：

- 将 `base_url` 替换为您的实际 Visionatrix 服务器 URL（如果不同）。
- 对于第二个示例，将 `input_image_path` 替换为输入图像文件的路径。确保该文件存在于指定路径。
- 创建任务时，API 期望一个 `PUT` 请求使用 `multipart/form-data`。因此，即使您没有上传任何文件（如在 `sdxl_lighting` 流程中），您也应该使用 `files=params` 来确保请求使用正确的 content type。
- 这些脚本包含基本的错误处理和轮询逻辑。
- `check_task_progress` 函数每 5 秒轮询一次任务进度。根据需要调整睡眠时间。
- `retrieve_task_results` 函数使用从 `check_task_progress` 函数获得的 `outputs`，避免了对服务器的额外调用。
- 结果文件保存在当前工作目录中，文件名包含任务 ID 和节点 ID。
- 确保在运行这些脚本之前，您的 Visionatrix 实例上已安装所需的流程（`sdxl_lighting` 和 `remove_background_birefnet`）。
- 其他可选参数如 `webhook_url`、`translate`、`group_scope`、`seed` 等可用，但不在初学者指南中覆盖。

