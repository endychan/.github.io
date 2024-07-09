<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>HID Passthrough Tool</title>
    <style>
        * {
            margin: 0;
            padding: 0;
        }

        html,
        body {
            height: 100vh;
            background-color: #f7f7ff;
        }

        div {
            height: calc(100% - 4rem);
            padding: 2rem;
            display: grid;
            grid-template-columns: 1fr 1fr 1fr;
            grid-template-rows: 2rem 1fr;
            row-gap: 1rem;
            column-gap: 2rem;
        }

        div_child {
            display: grid;
            grid-template-columns: 1fr 1fr;
            grid-template-rows: 2rem 0fr;
            row-gap: 1rem;
            column-gap: 2rem;
        }

        textarea {
            resize: none;
            overflow-y: scroll;
            overflow-x: hidden;
            padding: 1rem;
        }
    </style>
    <script>
        function getBroswer() {
            var sys = {};
            var ua = navigator.userAgent.toLowerCase();
            var s;
            (s = ua.match(/edge\/([\d.]+)/)) ? sys.edge = s[1] :
                (s = ua.match(/rv:([\d.]+)\) like gecko/)) ? sys.ie = s[1] :
                    (s = ua.match(/msie ([\d.]+)/)) ? sys.ie = s[1] :
                        (s = ua.match(/firefox\/([\d.]+)/)) ? sys.firefox = s[1] :
                            (s = ua.match(/chrome\/([\d.]+)/)) ? sys.chrome = s[1] :
                                (s = ua.match(/opera.([\d.]+)/)) ? sys.opera = s[1] :
                                    (s = ua.match(/version\/([\d.]+).*safari/)) ? sys.safari = s[1] : 0;

            if (sys.edge) return { broswer: "Edge", version: sys.edge };
            if (sys.ie) return { broswer: "IE", version: sys.ie };
            if (sys.firefox) return { broswer: "Firefox", version: sys.firefox };
            if (sys.chrome) return { broswer: "Chrome", version: sys.chrome };
            if (sys.opera) return { broswer: "Opera", version: sys.opera };
            if (sys.safari) return { broswer: "Safari", version: sys.safari };

            return { broswer: "", version: "0" };
        }

        if (!('hid' in navigator)) {
            alert('当前浏览器不支持 Web HID API串口操作,请更换Edge或Chrome浏览器')
        } else {
            var abc = getBroswer();
            alert("broswer:" + abc.broswer + " version:" + abc.version)
        }
    </script>
</head>

<body>
    <div>
        <div_child>
            <button id="btnOpen">open</button>
            <button id="btnClose" disabled="True">close</button>
        </div_child>
        <button id="btnSend">send</button>
        <button id="btnClear">clear</button>
        <textarea id="iptLog" readonly></textarea>
        <textarea id="iptOutput">7c ff ff 82 32 00 d2</textarea>
        <textarea id="iptInput" readonly></textarea>
    </div>
    <script>
        const btnOpen = document.querySelector("#btnOpen");
        const btnClose = document.querySelector("#btnClose");
        const btnSend = document.querySelector("#btnSend");
        const btnClear = document.querySelector("#btnClear");
        const iptLog = document.querySelector("#iptLog");
        const iptOutput = document.querySelector("#iptOutput");
        const iptInput = document.querySelector("#iptInput");

        iptLog.value += "HID Passthrough Tool\n\n";
        iptLog.value += "This is an HID Passthrough device read/write Tool.\n\n";
        iptLog.value += "Device must have one collection with one input and one output.\n\n";
        iptLog.value += "For more detail see below:\n\n";

        let device; // 需要连接或已连接的设备
        let inputDataLength; // 发送数据包长度
        let outputDataLength; // 发送数据包长度

        // 打开设备相关操作
        btnOpen.onclick = async () => {
            try {
                // requestDevice方法将显示一个包含已连接设备列表的对话框，用户选择可以并授予其中一个设备访问权限
                const devices = await navigator.hid.requestDevice({ filters: [{ vendorId: 1240, productId: 831 }] });

                // const devices = await navigator.hid.requestDevice({
                //     filters: [{
                //         vendorId: 0xabcd,  // 根据VID进行过滤
                //         productId: 0x1234, // 根据PID进行过滤
                //         usagePage: 0x0c,   // 根据usagePage进行过滤
                //         usage: 0x01,       // 根据usage进行过滤
                //     },],
                // });

                // let devices = await navigator.hid.getDevices(); // getDevices方法可以返回已连接的授权过的设备列表

                if (devices.length == 0) {
                    iptLog.value += "No device selected\n\n";
                    iptLog.scrollTop = iptLog.scrollHeight; // 滚动到底部
                    return;
                }

                device = devices[0]; // 选择列表中第一个设备

                if (!device.opened) {
                    // 检查设备是否打开
                    await device.open(); // 打开设备
                    document.getElementById('btnOpen').disabled = true;
                    document.getElementById('btnClose').disabled = false;
                    // 下面几行代码和我的自定义的透传的HID设备有关
                    // 我的设备中有一个collection，包含一个input、一个output
                    // inputReports和outputReports数据是Array，reportSize是8
                    // reportCount表示一包数据的字节数，USB-FS 和 USB-HS 设置的reportCount最大值不同
                    if (device.collections[0].inputReports[0].items[0].isArray && device.collections[0].inputReports[0].items[0].reportSize === 8) {

                        inputDataLength = device.collections[0].inputReports[0].items[0].reportCount ?? 0;
                    }
                    if (device.collections[0].outputReports[0].items[0].isArray && device.collections[0].outputReports[0].items[0].reportSize === 8) {
                        // 发送数据包长度必须和报告描述符中描述的一致
                        outputDataLength = device.collections[0].outputReports[0].items[0].reportCount ?? 0;
                    }

                    iptLog.value += `Open device: \n${device.productName}\nPID-${device.productId} VID-${device.vendorId}\ninputDataLength-${inputDataLength} outputDataLength-${outputDataLength}\n\n`;
                    iptLog.scrollTop = iptLog.scrollHeight; // 滚动到底部
                }
                // await device.close(); // 关闭设备
                // await device.forget() // 遗忘设备

                // 电脑接收到来自设备的消息回调
                device.oninputreport = (event) => {
                    console.log(event); // event中包含device、reportId、data等内容

                    let array = new Uint8Array(event.data.buffer); // event.data.buffer就是接收到的inputreport包数据了
                    let hexstr = "";
                    for (let i = 0; i < array[0]; i++) {
                        hexstr += (Array(2).join(0) + array[i + 1].toString(16).toUpperCase()).slice(-2) + " "; // 将字节数据转换成（XX ）形式字符串
                    }
                    //for (const data of array) {
                    //    hexstr += (Array(2).join(0) + data.toString(16).toUpperCase()).slice(-2) + " "; // 将字节数据转换成（XX ）形式字符串
                    //}
                    iptInput.value += hexstr;
                    iptInput.scrollTop = iptInput.scrollHeight; // 滚动到底部

                    iptLog.value += `Received ${event.data.byteLength} bytes\n\n`;
                    iptLog.scrollTop = iptLog.scrollHeight; // 滚动到底部
                };

            } catch (error) {
                iptLog.value += `${error}\n\n`;
                iptLog.scrollTop = iptLog.scrollHeight; // 滚动到底部
            }
        };

        // 关闭串口
        btnClose.onclick = async () => {
            if ((device === null) || (!device?.opened)) {
                console.log("Not opened.");
                return;
            }

            await device.close(); // 关闭设备
            await device.forget() // 遗忘设备
            device = null;
            document.getElementById('btnOpen').disabled = false;
            document.getElementById('btnClose').disabled = true;
        }

        // 发送数据相关操作
        btnSend.onclick = async () => {
            try {
                if (!device?.opened) {
                    throw "Device not opened";
                }

                const outputData = new Uint8Array(outputDataLength); // 要发送的数据包

                let outputDatastr = iptOutput.value.replace(/\s+/g, ""); // 去除所有空白字符
                if (outputDatastr.length % 2 == 0 && /^[0-9a-fA-F]+$/.test(outputDatastr)) {
                    // 检查长度和字符是否正确
                    // 一包长度不能大于报告描述符中规定的长度
                    const byteLength = outputDatastr.length / 2 > outputDataLength ? outputDataLength : outputDatastr.length / 2;
                    outputData[0] = byteLength;
                    // 将字符串转成字节数组数据
                    for (let i = 0; i < byteLength; i++) {
                        outputData[i + 1] = parseInt(outputDatastr.substr(i * 2, 2), 16);
                    }
                } else {
                    throw "Data is not even or 0-9、a-f、A-F";
                }

                await device.sendReport(0, outputData); // 发送数据，第一个参数为reportId，填0表示不使用reportId
                iptLog.value += `Send ${outputData.length} bytes\n\n`;
                iptLog.scrollTop = iptLog.scrollHeight; // 滚动到底部

            } catch (error) {
                iptLog.value += `${error}\n\n`;
                iptLog.scrollTop = iptLog.scrollHeight; // 滚动到底部
            }
        };

        // 全局HID设备插入事件
        navigator.hid.onconnect = (event) => {
            console.log("HID connected: ", event.device); // device 的 collections 可以看到设备报告描述符相关信息
            iptLog.value += `HID connected:\n${event.device.productName}\nPID ${event.device.productId} VID ${event.device.vendorId}\n\n`;
            iptLog.scrollTop = iptLog.scrollHeight; // 滚动到底部
        };

        // 全局HID设备拔出事件
        navigator.hid.ondisconnect = (event) => {
            device = null; // 释放当前设备
            iptLog.value += `HID disconnected:\n${event.device.productName}\nPID ${event.device.productId} VID ${event.device.vendorId}\n\n`;
            iptLog.scrollTop = iptLog.scrollHeight; // 滚动到底部
        };

        // 清空数据接收窗口
        btnClear.onclick = () => {
            iptInput.value = "";
        };
    </script>
</body>
</html>