# ntit-ses
南通理工学院融合门户学生评教脚本


标题瞎写的 懒得创项目了
这个md5加密好好笑

```py
import requests
import json
import time
import hashlib

# f12自己找
user_token = "your_token_here"
# 间隔
interval = 1

# 设置请求头
headers = {
    "accept": "application/json, text/plain, */*",
    "accept-encoding": "gzip, deflate, br, zstd",
    "accept-language": "en-US,en;q=0.9",
    "content-type": "application/json",
    "cookie": f"token={user_token}",
    "user-agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36"
}

# API URLs
list_url = f"https://zlpgxt.ntit.edu.cn/api/v2/pj/all/list?token={user_token}"
questionnaire_url_template = "https://zlpgxt.ntit.edu.cn/api/v2/questionnaire/paper/{xspjid}?token=" + user_token
submit_url = "https://zlpgxt.ntit.edu.cn/api/v2/questionnaire/paper/{xspjid}?token=" + user_token

# 获取课程列表
list_response = requests.get(list_url, headers=headers)
if list_response.status_code != 200:
    print("获取课程列表失败:", list_response.status_code, list_response.text)
    exit()

course_list = list_response.json().get("data", [])

# 遍历课程列表
for course in course_list:
    course_name = course.get("COURSENAME")
    teacher_name = course.get("TEACHERNAME")
    pkbh = course.get("PKBH")
    xspjid = course.get("XSPJID")

    print(f"正在处理课程: {course_name}, 教师: {teacher_name}, PKBH: {pkbh}")

    # 获取问卷题目
    questionnaire_url = questionnaire_url_template.format(xspjid=xspjid)
    questionnaire_response = requests.get(questionnaire_url, headers=headers)
    if questionnaire_response.status_code != 200:
        print(f"获取问卷失败: {questionnaire_response.status_code}, {questionnaire_response.text}")
        continue

    questionnaire_data = questionnaire_response.json()
    paper_subject_list = questionnaire_data.get("data", {}).get("paperSubjectList", [])

    # 构造答案
    answers = {}
    for subject in paper_subject_list:
        subject_id = subject.get("subjectId")
        if subject_id:
            answers[f"s{subject_id}"] = {"result": "A"}  # 使用 "A" (100分)

    # 构造提交数据
    post_data = {
        "status": "1",
        "basicParams": f"{pkbh}@11@",
        "answers": answers,
        "commentator": course.get("TEACHERNO"),
        "deviceId": "d7fb6df-6b00-1af",
        "param1": pkbh,
        "param2": "11",
        "param3": "1734318000000",  # 固定时间戳
        "param4": "20511096",
        "signature": "",
        "verification": "8bc2ff3c0ba897040a74b0e5a470e457"
    }

    # 动态生成验证字段 (示例算法，需根据实际规则调整)
    def generate_verification(queryBase64):
        """
        根据特定字段动态生成 verification
        """
        base_string = (
            queryBase64.get("commentator", "") +
            queryBase64.get("basicParams", "") +
            queryBase64.get("param1", "") +
            queryBase64.get("param2", "") +
            queryBase64.get("param3", "") +
            queryBase64.get("param4", "") +
            "murp2020"
        )
        return hashlib.md5(base_string.encode()).hexdigest()

    # 更新 verification
    queryBase64 = {
        "commentator": post_data["commentator"],
        "basicParams": post_data["basicParams"],
        "param1": post_data["param1"],
        "param2": post_data["param2"],
        "param3": post_data["param3"],
        "param4": post_data["param4"]
    }

    post_data["verification"] = generate_verification(queryBase64)

    # 测试 打印提交数据
    submit_response = requests.post(submit_url.format(xspjid=xspjid), headers=headers, data=json.dumps(post_data))
    if submit_response.status_code == 200:
        print(f"提交成功: {course_name}, 响应: {submit_response.json()}")
    else:
        print(f"提交失败: {course_name}, 状态码: {submit_response.status_code}, 错误: {submit_response.text}")
        # 输出所有数据
        # 写入log 全部数据
        with open("error.log", "a") as f:
            f.write(
                f"提交失败: {course_name}, 状态码: {submit_response.status_code}, 错误: {submit_response.text}, 数据: {json.dumps(post_data)}\n"
            )

    # 休息 1 秒
    time.sleep(interval)
    # break
```
