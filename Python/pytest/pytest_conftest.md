### 回调函数：获取用例执行结果

> https://www.jianshu.com/p/2637fb7bfbea

```python
# conftest.py 

import pytest

@pytest.hookimpl(hookwrapper=True, tryfirst=True)
def pytest_runtest_makereport(item, call):
    print('------------------------------------')

    # 获取钩子方法的调用结果
    out = yield
    # print('用例执行结果', out)

    # 3\. 从钩子方法的调用结果中获取测试报告
    report = out.get_result()

    # print('测试报告：%s' % report)
    print('步骤：%s' % report.when)
    print('nodeid：%s' % report.nodeid)
    # print('description:%s' % str(item.function.__doc__))
    print(('运行结果: %s' % report.outcome))
```

运行结果：

```python
.------------------------------------
用例执行结果 <pluggy.callers._Result object at 0x0000027C54750A20>
测试报告：<TestReport 'test_a.py::test_a' when='teardown' outcome='passed'>
步骤：teardown
nodeid：test_a.py::test_a
description:用例描述:test_a
运行结果: passed
```



---

### 打印用例名【方法名】和执行结果

```python
from _pytest.runner import runtestprotocol

def pytest_runtest_protocol(item, nextitem):
    reports = runtestprotocol(item, nextitem=nextitem)
    for report in reports:
     if report.when == 'call':
      print('\n%s --- %s' % (item.name, report.outcome) )
    return True
```

运行结果：

```python
test_01 --- passed
```



---

### 用例的前后置操作

```python
# conftest.py 

# setup前置和teardown后置操作
@pytest.fixture(scope="function", autouse=True)
def fix_a():
    print("========== 用例开始执行 ========== ")
    yield
    # print("========== 恭喜，清空环境、执行用例、回收环境 都成功啦~ ========== ")
    # print(setup_num)
    print("\n\n\n")
    print("========== 用例执行结束 ========== ")
    print("\n")
```





---

### log的html格式设置

```python
# pytest -s --html=log.html

import time
from datetime import datetime
from lib2to3.pgen2 import driver
from py.xml import html

# 用例html的格式设置  ======================================================================
def pytest_configure(config):
    # 添加接口地址与项目名称
    config._metadata["设备IP"] = baseinfo.gwManageIp
    config._metadata['用例执行时间'] = str(time.strftime("%Y-%m-%d %H:%M:%S", time.localtime()))
    # 删除Java_Home
    config._metadata.pop("JAVA_HOME")


@pytest.mark.optionalhook
def pytest_html_results_table_header(cells):
    cells.insert(1, html.th('Time', class_='sortable time', col='time'))
    cells.insert(2, html.th('Description'))
    cells.pop(-1)  # 删除link列

@pytest.mark.optionalhook
def pytest_html_results_table_row(report, cells):
    cells.insert(1, html.td(datetime.utcnow(), class_='col-time'))
    cells.insert(2, html.td(report.description))
    cells.pop(-1)  # 删除link列
    # if report.passed:
    #   del cells[:]    # 如果用例passed，删除所有单元格


@pytest.mark.optionalhook
def pytest_html_results_summary(prefix):
    prefix.extend([html.p("所属部门: 卓讯-合肥测试部")])
    prefix.extend([html.p("测试人员: 王谦")])

def _capture_screenshot():
    '''
    截图保存为base64，展示到html中
    :return:
    '''
    return driver.get_screenshot_as_base64()

@pytest.mark.hookwrapper
def pytest_runtest_makereport(item):
    """
    当测试失败的时候，自动截图，展示到html报告中
    :param item:
    """
    pytest_html = item.config.pluginmanager.getplugin('html')
    outcome = yield
    report = outcome.get_result()
    extra = getattr(report, 'extra', [])

    if report.when == 'call' or report.when == "setup":
        xfail = hasattr(report, 'wasxfail')
        if (report.skipped and xfail) or (report.failed and not xfail):
            file_name = report.nodeid.replace("::", "_")+".png"
            screen_img = _capture_screenshot()
            if file_name:
                html = '<div><img src="data:image/png;base64,%s" alt="screenshot" style="width:600px;height:300px;" ' \
                       'onclick="window.open(this.src)" align="right"/></div>' % screen_img
                extra.append(pytest_html.extras.html(html))
        report.extra = extra
        report.description = str(item.function.__doc__)

@pytest.mark.optionalhook
def pytest_html_results_table_html(report, data):
    if report.skipped:  # 如果用例是skipped，则删除表格数据
        del data[:]
        data.append(html.div('No log output captured.', class_='empty log'))

```





