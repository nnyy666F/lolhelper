import os
import shutil
import subprocess
import sys
import logging
from PySide6.QtWidgets import QApplication, QWidget, QVBoxLayout, QHBoxLayout, QPushButton, QLabel, QLineEdit, QCheckBox, QGroupBox, QFileDialog, QListWidget, QMessageBox, QInputDialog
from PySide6.QtGui import QFont
from PySide6.QtCore import QThread, Signal
import time
import psutil

# 设置日志记录器
logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

class WegameSearchThread(QThread):
    def run(self):
        try:
            # 获取 fd.exe 的路径
            fd_path = os.path.join(os.path.dirname(__file__), 'fd.exe')
            logger.debug(f"Running command: {fd_path} WeGameApps -H")
            # 检查是否有有效的盘符输入
            if hasattr(self, 'drive') and self.drive:
                command = f'{fd_path} 英雄联盟 {self.drive}:\\'
                # 先设置控制台编码为 utf-8
                subprocess.run('chcp 936', shell=True)
                result = subprocess.run(command, shell=True, capture_output=True, text=True)
                if result.stdout:
                    # 判断结果是否为字节串，如果是则解码，否则直接使用
                    if isinstance(result.stdout, bytes):
                        decoded_stdout = result.stdout.decode('utf-8', errors='replace')
                        paths = decoded_stdout.splitlines()
                    else:
                        # 如果不是字节串，尝试将其转换为字符串并解码
                        temp_str = str(result.stdout)
                        decoded_stdout = temp_str.decode('utf-8', errors='replace') if hasattr(temp_str, 'decode') else temp_str
                        paths = decoded_stdout.splitlines()
                    # 将结果保存到文件
                    with open('search_results.txt', 'w', encoding='gbk') as f:
                        f.write('\n'.join(paths))
                    self.paths = paths
                else:
                    self.paths = []
            else:
                self.paths = []
            logger.debug(f"Found paths: {self.paths}")
        except Exception as e:
            logger.error(f"Error in WegameSearchThread: {e}")

class LeagueOfLegendsFixer(QWidget):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.initUI()
        self.search_thread = WegameSearchThread()

    def initUI(self):
        font = QFont()
        font.setPointSize(12)

        mainLayout = QHBoxLayout()

        instructionsLayout = QVBoxLayout()
        self.instructionsList = QListWidget()
        self.instructionsList.addItem("1. 首先定位 Wegame 路径。")
        self.instructionsList.addItem("2. 选择要移除的内容文件夹：")
        self.instructionsList.addItem("   - cross：可能与游戏中的跨服功能相关的文件夹，移除后可能影响跨服游戏体验。")
        self.instructionsList.addItem("   - wegamelauncher：Wegame 的启动相关文件夹，移除后可能影响Wegame的正常启动和更新。")
        self.instructionsList.addItem("   - feedbak：可能与游戏反馈功能相关的文件夹，移除后可能无法正常提交游戏反馈。")
        self.instructionsList.addItem("   - tphelper.exe：腾讯tp文件夹，移除后可能降低游戏的安全性。")
        self.instructionsList.addItem("   - yxq：迅游加速器的文件夹。")
        self.instructionsList.addItem("3. 点击执行进行卡顿修复。")
        self.instructionsList.addItem("4. 如需还原，选择备份路径后点击还原按钮。")

        instructionsLayout.addWidget(self.instructionsList)
        mainLayout.addLayout(instructionsLayout)

        rightLayout = QVBoxLayout()

        wegameLayout = QHBoxLayout()
        self.wegamePathLabel = QLabel("Wegame 路径：")
        self.wegamePathLabel.setFont(font)
        self.wegamePathInput = QLineEdit()
        self.wegamePathInput.setFont(font)
        self.browseWegameButton = QPushButton("浏览")
        self.browseWegameButton.setFont(font)
        self.browseWegameButton.clicked.connect(self.browseWegamePath)
        self.locateWegameButton = QPushButton("定位")
        self.locateWegameButton.setFont(font)
        self.locateWegameButton.clicked.connect(self.setWegamePathFromSearch)
        wegameLayout.addWidget(self.wegamePathLabel)
        wegameLayout.addWidget(self.wegamePathInput)
        wegameLayout.addWidget(self.browseWegameButton)
        wegameLayout.addWidget(self.locateWegameButton)
        rightLayout.addLayout(wegameLayout)

        self.timeLabel = QLabel("操作时间：")
        self.timeLabel.setFont(font)
        rightLayout.addWidget(self.timeLabel)

        restoreLayout = QHBoxLayout()
        self.restoreButton = QPushButton("还原")
        self.restoreButton.setFont(font)
        self.restoreButton.clicked.connect(self.performRestore)
        self.backupPathLabel = QLabel("备份路径：")
        self.backupPathLabel.setFont(font)
        self.backupPathInput = QLineEdit()
        self.backupPathInput.setFont(font)
        self.browseBackupButton = QPushButton("浏览")
        self.browseBackupButton.setFont(font)
        self.browseBackupButton.clicked.connect(self.browseBackupPath)
        restoreLayout.addWidget(self.restoreButton)
        restoreLayout.addWidget(self.backupPathLabel)
        restoreLayout.addWidget(self.backupPathInput)
        restoreLayout.addWidget(self.browseBackupButton)
        rightLayout.addLayout(restoreLayout)

        checkboxesLayout = QVBoxLayout()
        checkboxGroup = QGroupBox("选择移除的内容文件夹")
        self.crossCheckbox = QCheckBox("cross")
        self.wegamelauncherCheckbox = QCheckBox("wegamelauncher")
        self.feedbackCheckbox = QCheckBox("feedbak")
        self.tphelperCheckbox = QCheckBox("tphelper.exe")
        self.yxqCheckbox = QCheckBox("yxq")
        self.crossCheckbox.setChecked(True)
        self.wegamelauncherCheckbox.setChecked(True)
        self.feedbackCheckbox.setChecked(True)
        self.tphelperCheckbox.setChecked(True)
        self.yxqCheckbox.setChecked(True)
        checkboxesLayout.addWidget(self.crossCheckbox)
        checkboxesLayout.addWidget(self.wegamelauncherCheckbox)
        checkboxesLayout.addWidget(self.feedbackCheckbox)
        checkboxesLayout.addWidget(self.tphelperCheckbox)
        checkboxesLayout.addWidget(self.yxqCheckbox)
        checkboxGroup.setLayout(checkboxesLayout)
        rightLayout.addWidget(checkboxGroup)

        self.executeButton = QPushButton("执行")
        self.executeButton.setFont(font)
        self.executeButton.clicked.connect(self.performFix)
        rightLayout.addWidget(self.executeButton)

        # 添加启动英雄联盟按钮
        self.startLolButton = QPushButton("启动英雄联盟")
        self.startLolButton.setFont(font)
        self.startLolButton.clicked.connect(self.startLol)
        rightLayout.addWidget(self.startLolButton)

        mainLayout.addLayout(rightLayout)

        self.setLayout(mainLayout)
        self.setWindowTitle("英雄联盟卡顿修复工具")

    def browseWegamePath(self):
        try:
            filePath, _ = QFileDialog.getExistingDirectory(self, "选择 Wegame 路径", "")
            if filePath:
                self.wegamePathInput.setText(filePath)
        except Exception as e:
            logger.error(f"Error in browseWegamePath: {e}")

    def setWegamePathFromSearch(self):
        drive, ok = QInputDialog.getText(self, "选择盘符", "请输入盘符（如 C、D 等）：")
        if ok and drive:
            self.search_thread.drive = drive
            self.search_thread.start()
            msg_box = QMessageBox(self)
            msg_box.setText("正在定位，请稍候...")
            msg_box.setWindowTitle("定位中")
            msg_box.show()
            self.search_thread.finished.connect(lambda: self.showPathSelectionDialog(msg_box))
        else:
            pass

    def showPathSelectionDialog(self, msg_box):
        if os.path.exists('search_results.txt'):
            with open('search_results.txt', 'r', encoding='utf-8') as f:
                paths = f.read().splitlines()
            if paths:
                selection_dialog = QInputDialog(self)
                selection_dialog.setComboBoxItems(paths)
                selection_dialog.setWindowTitle("选择路径")
                selection_dialog.setLabelText("请选择路径：")
                ok = selection_dialog.exec()
                if ok:
                    selected_path = selection_dialog.textValue()
                    self.wegamePathInput.setText(selected_path)
                msg_box.close()
            else:
                logger.warning("No paths found.")
                QMessageBox.warning(self, "未找到路径", "未找到英雄联盟相关路径。")
                msg_box.close()
        else:
            logger.warning("No search results file found.")
            QMessageBox.warning(self, "未找到路径", "未找到英雄联盟相关路径。")
            msg_box.close()

    def browseBackupPath(self):
        try:
            filePath, _ = QFileDialog.getExistingDirectory(self, "选择备份路径", "")
            if filePath:
                self.backupPathInput.setText(filePath)
        except Exception as e:
            logger.error(f"Error in browseBackupPath: {e}")

    def performRestore(self):
        start_time = time.time()
        backup_path = self.backupPathInput.text()
        if not backup_path:
            desktop = os.path.join(os.path.expanduser("~"), "Desktop")
            backup_path = os.path.join(desktop, "LolBackup")
        wegame_path = self.wegamePathInput.text()
        items_to_restore = []
        if self.crossCheckbox.isChecked():
            cross_backup_path = os.path.join(backup_path, "cross")
            if os.path.exists(cross_backup_path):
                cross_target_path = os.path.join(wegame_path, "cross")
                if os.path.exists(cross_target_path):
                    try:
                        command = f'attrib -r "{cross_target_path}"'
                        subprocess.run(command, shell=True)
                        shutil.rmtree(cross_target_path)
                    except Exception as e:
                        QMessageBox.warning(self, "错误", f"处理 Wegame 路径下的 cross 文件夹时出错：{e}")
                shutil.move(cross_backup_path, cross_target_path)
                items_to_restore.append(cross_target_path)
            else:
                QMessageBox.warning(self, "警告", "备份中的 cross 文件夹不存在，无法还原。")
        if self.wegamelauncherCheckbox.isChecked():
            wegamelauncher_backup_path = os.path.join(backup_path, "wegamelauncher")
            if os.path.exists(wegamelauncher_backup_path):
                wegamelauncher_target_path = os.path.join(wegame_path, "wegamelauncher")
                if os.path.exists(wegamelauncher_target_path):
                    try:
                        command = f'attrib -r "{wegamelauncher_target_path}"'
                        subprocess.run(command, shell=True)
                        shutil.rmtree(wegamelauncher_target_path)
                    except Exception as e:
                        QMessageBox.warning(self, "错误", f"处理 Wegame 路径下的 wegamelauncher 文件夹时出错：{e}")
                    shutil.move(wegamelauncher_backup_path, wegamelauncher_target_path)
                    items_to_restore.append(wegamelauncher_target_path)
                else:
                    QMessageBox.warning(self, "警告", "备份中的 wegamelauncher 文件夹不存在，无法还原。")
            if self.feedbackCheckbox.isChecked():
                feedback_backup_path = os.path.join(backup_path, "FeedBack")
                if os.path.exists(feedback_backup_path):
                    feedback_target_path = os.path.join(wegame_path, "LeagueClient", "FeedBack")
                    if os.path.exists(feedback_target_path):
                        try:
                            command = f'attrib -r "{feedback_target_path}"'
                            subprocess.run(command, shell=True)
                            shutil.rmtree(feedback_target_path)
                        except Exception as e:
                            QMessageBox.warning(self, "错误", f"处理 Wegame 路径下的 FeedBack 文件夹时出错：{e}")
                    shutil.move(feedback_backup_path, feedback_target_path)
                    items_to_restore.append(feedback_target_path)
                else:
                    QMessageBox.warning(self, "警告", "备份中的 FeedBack 文件夹不存在，无法还原。")
            if self.tphelperCheckbox.isChecked():
                tp_backup_path = os.path.join(backup_path, "TP")
                if os.path.exists(tp_backup_path):
                    tp_target_path = os.path.join(wegame_path, "TCLS", "TenProtect", "TP")
                    if os.path.exists(tp_target_path):
                        try:
                            command = f'attrib -r "{tp_target_path}"'
                            subprocess.run(command, shell=True)
                            shutil.rmtree(tp_target_path)
                        except Exception as e:
                            QMessageBox.warning(self, "错误", f"处理 Wegame 路径下的 TP 文件夹时出错：{e}")
                    shutil.move(tp_backup_path, tp_target_path)
                    items_to_restore.append(tp_target_path)
                else:
                    QMessageBox.warning(self, "警告", "备份中的 TP 文件夹不存在，无法还原。")
            if self.yxqCheckbox.isChecked():
                yxq_backup_path = os.path.join(backup_path, "yxq_nethelper")
                if os.path.exists(yxq_backup_path):
                    yxq_target_path = "C:\\Program Files (x86)\\yxq_nethelper"
                    if os.path.exists(yxq_target_path):
                        try:
                            command = f'attrib -r "{yxq_target_path}"'
                            subprocess.run(command, shell=True)
                            shutil.rmtree(yxq_target_path)
                        except Exception as e:
                            QMessageBox.warning(self, "错误", f"处理 yxq_nethelper 文件夹时出错：{e}")
                    shutil.move(yxq_backup_path, yxq_target_path)
                    items_to_restore.append(yxq_target_path)
                else:
                    QMessageBox.warning(self, "警告", "备份中的 yxq_nethelper 文件夹不存在，无法还原。")
            # 解除只读属性
            for item in items_to_restore:
                if os.path.isdir(item):
                    try:
                        command = f'attrib -r "{item}"'
                        subprocess.run(command, shell=True)
                    except Exception as e:
                        QMessageBox.warning(self, "错误", f"解除 {item} 的只读模式时出错：{e}")                       
            end_time = time.time()
            elapsed_time = end_time - start_time
            self.timeLabel.setText(f"操作时间：{elapsed_time:.2f} 秒")



    def performFix(self):
        start_time = time.time()
        wegame_path = self.wegamePathInput.text()
        backup_path = self.backupPathInput.text()
        if not backup_path:
            desktop = os.path.join(os.path.expanduser("~"), "Desktop")
            backup_path = os.path.join(desktop, "LolBackup")
        items_to_move = []

        # 检查 LeagueClient.exe 和 wegame.exe 是否正在运行
        league_client_running = False
        wegame_running = False
        for proc in psutil.process_iter(['name']):
            if proc.info['name'] == 'LeagueClient.exe':
                league_client_running = True
            elif proc.info['name'] == 'wegame.exe':
                wegame_running = True
                #choice = QMessageBox.question(self, "提示", "LeagueClient.exe 和 wegame.exe 正在运行，是否停止这些进程以进行操作？", QMessageBox.Yes | QMessageBox.No)
        # if choice == QMessageBox.Yes:
        #     league_client_ux_running = False
        #     league_client_ux_render_running = False
        #     wegame_running = False
        #     for proc in psutil.process_iter(['name']):
        #         if proc.info['name'] == 'LeagueClientUx.exe':
        #             league_client_ux_running = True
        #         elif proc.info['name'] == 'LeagueClientUxRender.exe':
        #             league_client_ux_render_running = True
        #         elif proc.info['name'] == 'wegame.exe':
        #             wegame_running = True
        #     if wegame_running and (not league_client_ux_running or not league_client_ux_render_running):
        #         QMessageBox.warning(self, "提示", "请等到大厅的时候再执行，LeagueClientUx.exe 或 LeagueClientUxRender.exe 正在加载中。")
        #         return
        #     for proc in psutil.process_iter(['name']):
        #         if proc.info['name'] in ['LeagueClientUxRender.exe', 'wegame.exe']:
        #             proc.kill()
        # else:
        #     pass
        if league_client_running or wegame_running:
            QMessageBox.warning(self, "警告", "LeagueClient.exe 和 wegame.exe 正在运行，请手动停止这些进程以后在进行操作")
            return

        if self.crossCheckbox.isChecked():
            cross_path = os.path.join(wegame_path, "cross")
            if os.path.exists(cross_path):
                backup_cross_path = os.path.join(backup_path, "cross")
                if not os.path.exists(backup_cross_path):
                    os.makedirs(backup_cross_path)
                for item in os.listdir(cross_path):
                    src_item = os.path.join(cross_path, item)
                    dst_item = os.path.join(backup_cross_path, item)
                    shutil.move(src_item, dst_item)
                # 设置 cross 文件夹为只读
                command = f'attrib +r "{cross_path}"'
                subprocess.run(command, shell=True)
                items_to_move.append(cross_path)
            else:
                QMessageBox.warning(self, "警告", "cross 文件夹不存在，无法移除。")
        if self.wegamelauncherCheckbox.isChecked():
            wegamelauncher_path = os.path.join(wegame_path, "wegamelauncher")
            if os.path.exists(wegamelauncher_path):
                backup_wegamelauncher_path = os.path.join(backup_path, "wegamelauncher")
                if not os.path.exists(backup_wegamelauncher_path):
                    os.makedirs(backup_wegamelauncher_path)
                for item in os.listdir(wegamelauncher_path):
                    src_item = os.path.join(wegamelauncher_path, item)
                    dst_item = os.path.join(backup_wegamelauncher_path, item)
                    shutil.move(src_item, dst_item)
                # 设置 wegamelauncher 文件夹为只读
                command = f'attrib +r "{wegamelauncher_path}"'
                subprocess.run(command, shell=True)
                items_to_move.append(wegamelauncher_path)
            else:
                QMessageBox.warning(self, "警告", "wegamelauncher 文件夹不存在，无法移除。")
        if self.feedbackCheckbox.isChecked():
            feedback_path = os.path.join(wegame_path, "LeagueClient", "FeedBack")
            if os.path.exists(feedback_path):
                backup_feedback_path = os.path.join(backup_path, "FeedBack")
                if not os.path.exists(backup_feedback_path):
                    os.makedirs(backup_feedback_path)
                for item in os.listdir(feedback_path):
                    src_item = os.path.join(feedback_path, item)
                    dst_item = os.path.join(backup_feedback_path, item)
                    shutil.move(src_item, dst_item)
                # 设置 feedback 文件夹为只读
                command = f'attrib +r "{feedback_path}"'
                subprocess.run(command, shell=True)
                items_to_move.append(feedback_path)
            else:
                QMessageBox.warning(self, "警告", "FeedBack 文件夹不存在，无法移除。")
        if self.tphelperCheckbox.isChecked():
            tp_path = os.path.join(wegame_path, "TCLS", "TenProtect", "TP")
            if os.path.exists(tp_path):
                backup_tp_path = os.path.join(backup_path, "TP")
                if not os.path.exists(backup_tp_path):
                    os.makedirs(backup_tp_path)
                for file_name in os.listdir(tp_path):
                    item_path = os.path.join(tp_path, file_name)
                    backup_item_path = os.path.join(backup_tp_path, file_name)
                    shutil.move(item_path, backup_item_path)
                # 设置 TP 文件夹为只读
                command = f'attrib +r "{tp_path}"'
                subprocess.run(command, shell=True)
                items_to_move.append(tp_path)
            else:
                QMessageBox.warning(self, "警告", "TP 文件夹不存在，无法移除。")
        if self.yxqCheckbox.isChecked():
            yxq_path = "C:\\Program Files (x86)\\yxq_nethelper"
            if os.path.exists(yxq_path):
                backup_yxq_path = os.path.join(backup_path, "yxq_nethelper")
                if not os.path.exists(backup_yxq_path):
                    os.makedirs(backup_yxq_path)
                for item in os.listdir(yxq_path):
                    src_item = os.path.join(yxq_path, item)
                    dst_item = os.path.join(backup_yxq_path, item)
                    shutil.move(src_item, dst_item)
                # 设置 yxq_nethelper 文件夹为只读
                command = f'attrib +r "{yxq_path}"'
                subprocess.run(command, shell=True)
                items_to_move.append(yxq_path)
            else:
                QMessageBox.warning(self, "警告", "yxq_nethelper 文件夹不存在，无法移除。")
        end_time = time.time()
        elapsed_time = end_time - start_time
        self.timeLabel.setText(f"操作时间：{elapsed_time:.2f} 秒")

    def startLol(self):
        wegame_path = self.wegamePathInput.text()
        if wegame_path:
            if os.path.exists('search_results.txt'):
                with open('search_results.txt', 'r', encoding='utf-8') as f:
                    paths = f.read().splitlines()
                if paths:
                    selection_dialog = QInputDialog(self)
                    selection_dialog.setComboBoxItems(["启动 Wegame", "启动英雄联盟客户端"])
                    selection_dialog.setWindowTitle("选择启动方式")
                    selection_dialog.setLabelText("请选择要启动的程序：")
                    ok = selection_dialog.exec()
                    if ok:
                        selected_option = selection_dialog.textValue()
                        if selected_option == "启动 Wegame":
                            wegamelauncher_path = os.path.join(wegame_path, "wegamelauncher")
                            launcher_path = os.path.join(wegamelauncher_path, "launcher.exe")
                            bugreport_path = os.path.join(wegamelauncher_path, "bugreport.exe")
                            if os.path.exists(launcher_path) and os.path.exists(bugreport_path):
                                try:
                                    subprocess.Popen(launcher_path)
                                except Exception as e:
                                    QMessageBox.warning(self, "错误", f"启动 Wegame 时出错：{e}")
                            else:
                                QMessageBox.warning(self, "错误", "无法找到 Wegame 启动文件。")
                        elif selected_option == "启动英雄联盟客户端":
                            client_path = os.path.join(wegame_path, "TCLS", "client.exe")
                            if os.path.exists(client_path):
                                try:
                                    subprocess.Popen(client_path)
                                    start_time = time.time()
                                    while True:
                                        time.sleep(1)
                                        if time.time() - start_time > 10:
                                            # 10 秒后检查进程是否存在
                                            process_running = False
                                            for proc in psutil.process_iter(['name']):
                                                if proc.info['name'] == 'client.exe':
                                                    process_running = True
                                                    break
                                            if not process_running:
                                                QMessageBox.warning(self, "错误", "英雄联盟启动失败。")
                                                choice = QMessageBox.question(self, "选择", "是否进行深度启动？", QMessageBox.Yes | QMessageBox.No)
                                                if choice == QMessageBox.Yes:
                                                    # 停止可能存在的进程
                                                    for proc in psutil.process_iter(['name']):
                                                        if proc.info['name'] == 'client.exe':
                                                            proc.kill()
                                                    # 再次启动
                                                    subprocess.Popen(client_path)
                                                else:
                                                    break
                                            else:
                                                break
                                except Exception as e:
                                    QMessageBox.warning(self, "错误", f"启动英雄联盟时出错：{e}")
                            else:
                                QMessageBox.warning(self, "错误", "无法找到英雄联盟客户端可执行文件。")
                    else:
                        pass
                else:
                    QMessageBox.warning(self, "未找到路径", "未找到英雄联盟相关路径。")
            else:
                QMessageBox.warning(self, "未找到路径", "未找到英雄联盟相关路径。")
        else:
            QMessageBox.warning(self, "错误", "请先设置 Wegame 路径。")



if __name__ == "__main__":
    try:
        app = QApplication([])
        window = LeagueOfLegendsFixer()
        logger.debug("Showing window")
        window.show()
        sys.exit(app.exec())
    except Exception as e:
        logger.error(f"Error in main: {e}")
