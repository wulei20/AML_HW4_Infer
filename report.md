# AML HW4 Infer
本次作业由于时间不足仅跑通了baseline并完成了实验报告。

## 程序实现
### 程序配置
* 任务设置：本实验模型目标为更准确地解答数学问题。
* 模型选择：本实验的基础模型与评判模型均直接采用了baseline的ZhipuAI提供的glm-4-flash模型。
* 验证数据集：本实验直接采用了baseline对math500截取的前10个问题进行测试。
* 环境配置：见 `requirements.txt` 文件。

### 算法实现
针对原始模型的优化算法存放于 `o1/g1` 文件中，其核心是 `generate_g1_response` 方法，它通过多个步骤逐步生成对用户问题的响应。以下详细讲解该文件的实现细节。

#### `analyze_response`
* 辅助方法，用来向 `glm_response` 方法发送请求，并解析返回值。
* 参数：
    * `messages`：对话上下文。
    * `model`：使用的模型（默认是 `glm-4-flash`）。
    * `is_final_answer`：是否请求最终答案。
* 功能：
    * 如果 `is_final_answer` 为 `True`，直接返回模型的响应。
    * 否则：
        * 优先尝试将响应解析为 JSON 格式。
        * 若解析失败，则通过正则表达式从返回的 Markdown 代码块中提取 JSON 并尝试解析。
        * 如果所有尝试都失败，返回一个错误消息。
    * 提供了 5 次重试机制，确保响应的稳定性。

#### `generate_g1_response`
* 核心方法，用于逐步生成推理链：
    1. 构建初始消息列表，包括系统角色说明和用户的 `prompt`。
    2. 使用 `analyze_response` 方法逐步获取每一步的推理结果。
        * 每一步都会更新 `messages`，确保模型有完整的上下文。
        * 如果收到 `final_answer` 的指令，停止迭代。
        * 限制最大步数为 25，防止进入死循环。
    3. 最后发送一条请求，获取最终答案。
    4. 返回完整的步骤列表、总耗时、以及最终答案。

#### 核心逻辑
核心逻辑包括：
* 定义问题（prompt）。
* 逐步构建推理链，最终生成可解释的回答。
* 输出步骤详情和最终答案。

具体细节部分还进行了如下处理：
* 递归推理：通过 `analyze_response` 方法的 JSON 响应中包含的 `next_action` 字段，判断是否继续下一步推理。
* 容错性：
    * JSON 格式的多重解析（直接解析或从代码块提取）。
    * 针对错误的 5 次重试机制。
* 性能监控：记录每一步的耗时，最终返回总时间。

### 模型评判
将从自定义算法中获取的结果与参考答案同时发送给 `critic_model` 进行四次评判，只要有一次评判结果得分为 `1`，则认为该问题回答正确。

## 结果分析
### 实验截图
![实验截图](img/result.png)

### 性能评估
推理结果见 `evaluate/evaluate_result_g1.jsonl` 文件，其中包含了每个问题的ID、正确性、推测答案。
* 结果准确性：在对10个问题的测试中，本次实验的准确率为 `70%`。
* 推理速度：10个问题总共用时30:45，平均每个问题用时约3分钟。但考虑到网络延迟，实际推理时间应该更短。

#### 推理示例
"problem": "Convert the point $(0,3)$ in rectangular coordinates to polar coordinates.  Enter your answer in the form $(r,\theta),$ where $r > 0$ and $0 \le \theta < 2 \pi.$"

Step 1: {'title': 'Identifying Key Information', 'content': 'We are given a point in rectangular coordinates (0,3) and need to find its representation in polar coordinates. Polar coordinates are represented as (r, θ), where r is the distance from the origin and θ is the angle in radians measured from the positive x-axis. We need to find r using the Pythagorean theorem and θ using the arctangent function.', 'next_action': 'continue'} 

Step 2: {'title': 'Identifying Key Information', 'content': 'We are given a point in rectangular coordinates, $(0,3)$, and we need to convert it to polar coordinates. The polar coordinates are represented as $(r,\theta)$, where $r$ is the distance from the origin to the point and $\theta$ is the angle formed with the positive x-axis. We need to find both $r$ and $\theta$.', 'next_action': 'continue'} 

Step 3: {'title': 'Calculating the Magnitude (r)', 'content': 'To find the magnitude r, we use the Pythagorean theorem, which states that in a right triangle, the square of the length of the hypotenuse (the side opposite the right angle) is equal to the sum of the squares of the lengths of the other two sides. In this case, the point (0,3) is on the y-axis, so the distance from the origin is just the y-coordinate, which is 3. Therefore, r = 3.', 'next_action': 'continue'} 

Step 4: {'title': 'Calculating the Radius (r)', 'content': 'The radius (r) in polar coordinates can be found using the Pythagorean theorem. Since the point is on the y-axis, the x-coordinate is 0. The distance from the origin is simply the y-coordinate, which is 3. Therefore, $r = 3$. We can confirm this by checking the Pythagorean theorem: $r^2 = x^2 + y^2$ which becomes $3^2 = 0^2 + 3^2 = 9$, which is correct.', 'next_action': 'continue'} 

Step 5: {'title': 'Calculating the Magnitude (r)', 'content': 'Given the rectangular coordinates (0, 3), the magnitude r in polar coordinates is the distance from the origin to the point. Since the point lies on the y-axis, the distance is simply the absolute value of the y-coordinate. Thus, r = 3.', 'next_action': 'continue'} 

Step 6: {'title': 'Determining the Angle (θ)', 'content': 'To find the angle θ, we use the fact that tan(θ) = opposite/adjacent. In this case, the opposite side is 3 (the y-coordinate) and the adjacent side is 0 (the x-coordinate). Since tan(θ) = 3/0, which is undefined, we must consider the special case where the point lies on the y-axis. The angle θ will be π/2 radians, which is 90 degrees, as the point is directly above the origin on the positive y-axis.', 'next_action': 'continue'} 

Step 7: {'title': 'Calculating the Angle (θ)', 'content': 'Now that we have found r = 3, we need to find the angle θ. Since the point lies on the y-axis, the angle with the positive x-axis will be either 0 or π (pi radians) because the x-coordinate is 0. However, since the y-coordinate is positive, the angle is π (pi radians). Therefore, θ = π.', 'next_action': 'continue'} 

Step 8: {'title': 'Determining the Angle (θ)', 'content': 'In polar coordinates, the angle θ is the angle formed with the positive x-axis. For the point (0,3), the angle θ is the angle from the positive x-axis to the point on the y-axis. Since the point is on the y-axis and above the origin, θ will be π/2 radians, which is equivalent to 90 degrees. This is because the tangent of π/2 is undefined, and the point lies in the second quadrant.', 'next_action': 'final_answer'} 

#### 错误示例
{"problem": "The results of a cross-country team's training run are graphed below. Which student has the greatest average speed? [asy]\nfor ( int i = 1; i <= 7; ++i )\n{\n\ndraw((i,0)--(i,6));\n}\n\nfor ( int i = 1; i <= 5; ++i )\n{\n\ndraw((0,i)--(8,i));\n}\ndraw((-0.5,0)--(8,0), linewidth(1));\ndraw((0,-0.5)--(0,6), linewidth(1));\nlabel(\"$O$\", (0,0), SW);\nlabel(scale(.85)*rotate(90)*\"distance\", (0, 3), W);\nlabel(scale(.85)*\"time\", (4, 0), S);\ndot((1.25, 4.5));\nlabel(scale(.85)*\"Evelyn\", (1.25, 4.8), N);\ndot((2.5, 2.2));\nlabel(scale(.85)*\"Briana\", (2.5, 2.2), S);\ndot((4.25,5.2));\nlabel(scale(.85)*\"Carla\", (4.25, 5.2), SE);\ndot((5.6, 2.8));\nlabel(scale(.85)*\"Debra\", (5.6, 2.8), N);\ndot((6.8, 1.4));\nlabel(scale(.85)*\"Angela\", (6.8, 1.4), E);\n[/asy]", "solution": "Evelyn covered more distance in less time than Briana, Debra and Angela, so her average speed is greater than any of their average speeds. Evelyn went almost as far as Carla in less than half the time that it took Carla, so Evelyn's average speed is also greater than Carla's. Therefore, $\\boxed{\\text{Evelyn}}$ is our answer.", "answer": "\\text{Evelyn}", "subject": "Algebra", "level": 2, "unique_id": "test/algebra/1349.json"}

{"ID": "test/algebra/1349.json", "Correct": false, "Predicted Answer": "6 miles per hour.", "Predicted": "Evelyn has the greatest average speed of 3.6 miles per hour."}

错误点分析：首先是答案在截取时误将小数点作为上一句话的结束，导致了答案的错误。其次是模型在推理时对图片的处理似乎并不好。