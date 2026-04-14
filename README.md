# yangh_music
I NEED
这只是一个线上转接笔记
保存为：music_terminal.py
#!/usr/bin/env python3
"""
🎵 终极终端音乐播放器 - 支持四卷合一
支持：顺序播放、随机播放、单曲循环、列表循环、区间播放
"""

import os
import sys
import time
import random
import threading
import readline  # 支持终端输入历史
from pathlib import Path

# 导入四卷
try:
    from my_music_box_vol1 import MusicBox as Box1
    from my_music_box_vol2 import MusicBox as Box2
    from my_music_box_vol3 import MusicBox as Box3
    from my_music_box_vol4 import MusicBox as Box4
except ImportError as e:
    print(f"❌ 导入失败: {e}")
    print("请先安装: pip install my-music-box-vol1 my-music-box-vol2 my-music-box-vol3 my-music-box-vol4")
    sys.exit(1)


class TerminalMusicPlayer:
    def __init__(self):
        self.playlist = []          # 全局播放列表
        self.current_index = 0      # 当前播放索引
        self.play_mode = "sequence" # sequence/random/single/loop
        self.is_playing = False
        self.is_paused = False
        self.current_box = None
        self.play_history = []      # 播放历史
        self.favorites = set()      # 收藏
        
        self._load_all_music()
        self._setup_shortcuts()
        
    def _load_all_music(self):
        """加载四卷音乐并建立全局索引"""
        print("🎵 正在加载音乐库...")
        
        boxes = [
            (1, Box1()), (2, Box2()), 
            (3, Box3()), (4, Box4())
        ]
        
        idx = 0
        for vol_num, box in boxes:
            for track in box.catalog:
                self.playlist.append({
                    'global_idx': idx,
                    'vol': vol_num,
                    'box': box,
                    'artist': track.get('artist', 'Unknown'),
                    'title': track.get('title', 'Unknown'),
                    'track_data': track
                })
                idx += 1
        
        print(f"✅ 加载完成！共 {len(self.playlist)} 首歌曲（4卷）\n")
        
    def _setup_shortcuts(self):
        """设置终端快捷键（如果系统支持）"""
        # 尝试设置终端标题
        os.system(f"title 🎵 音乐播放器 - {len(self.playlist)}首歌" if os.name == 'nt' else "")
    
    def show_library(self):
        """显示完整歌库"""
        print(f"\n{'='*80}")
        print(f"📚 音乐库清单 (共 {len(self.playlist)} 首)")
        print(f"{'='*80}")
        
        current_vol = 0
        for song in self.playlist:
            if song['vol'] != current_vol:
                current_vol = song['vol']
                print(f"\n【第 {current_vol} 卷】")
                print("-" * 80)
            
            marker = "♥" if song['global_idx'] in self.favorites else " "
            print(f"{marker} [{song['global_idx']:3d}] {song['artist']} - {song['title']}")
        
        print(f"{'='*80}\n")
    
    def search(self, keyword):
        """搜索歌曲"""
        keyword = keyword.lower()
        results = [
            song for song in self.playlist
            if keyword in song['artist'].lower() 
            or keyword in song['title'].lower()
        ]
        
        if not results:
            print(f"❌ 未找到 '{keyword}'")
            return []
        
        print(f"\n🔍 找到 {len(results)} 首包含 '{keyword}' 的歌曲:")
        print("-" * 80)
        for song in results[:20]:  # 最多显示20首
            print(f"  [{song['global_idx']:3d}] {song['artist']} - {song['title']}")
        print("-" * 80)
        
        return results
    
    def play_by_index(self, index):
        """播放指定索引的歌曲"""
        if index < 0 or index >= len(self.playlist):
            print(f"❌ 索引 {index} 超出范围 (0-{len(self.playlist)-1})")
            return
        
        song = self.playlist[index]
        self.current_index = index
        self.current_box = song['box']
        
        # 停止当前播放
        self.stop()
        
        print(f"\n▶️  [{index}] {song['artist']} - {song['title']} [第{song['vol']}卷]")
        
        # 在新线程中播放，避免阻塞终端
        self.is_playing = True
        self.is_paused = False
        self.play_history.append(index)
        
        threading.Thread(target=self._play_thread, args=(song,), daemon=True).start()
    
    def _play_thread(self, song):
        """播放线程"""
        try:
            song['box'].play(song['track_data'])
        except Exception as e:
            print(f"❌ 播放失败: {e}")
            self.is_playing = False
    
    def stop(self):
        """停止播放"""
        if self.current_box and self.is_playing:
            try:
                self.current_box.stop()
            except:
                pass
        self.is_playing = False
        self.is_paused = False
    
    def pause(self):
        """暂停/继续"""
        if not self.current_box or not self.is_playing:
            return
        
        if self.is_paused:
            try:
                self.current_box.resume()
                self.is_paused = False
                print("▶️  继续播放")
            except:
                pass
        else:
            try:
                self.current_box.pause()
                self.is_paused = True
                print("⏸️  已暂停")
            except:
                pass
    
    def next_song(self):
        """下一首（根据播放模式）"""
        if not self.playlist:
            return
        
        if self.play_mode == "random":
            self.current_index = random.randint(0, len(self.playlist) - 1)
        elif self.play_mode == "single":
            pass  # 单曲循环，索引不变
        else:  # sequence 或 loop
            self.current_index += 1
            if self.current_index >= len(self.playlist):
                if self.play_mode == "loop":
                    self.current_index = 0
                else:
                    print("\n🛑 播放列表已结束")
                    self.stop()
                    return
        
        self.play_by_index(self.current_index)
    
    def prev_song(self):
        """上一首"""
        if not self.playlist:
            return
        
        if self.play_mode == "random":
            if len(self.play_history) >= 2:
                self.current_index = self.play_history[-2]
        else:
            self.current_index -= 1
            if self.current_index < 0:
                self.current_index = len(self.playlist) - 1
        
        self.play_by_index(self.current_index)
    
    def play_range(self, start, end, mode="sequence"):
        """播放指定区间"""
        start = max(0, start)
        end = min(len(self.playlist) - 1, end)
        
        if start > end:
            print("❌ 起始索引不能大于结束索引")
            return
        
        print(f"\n🎵 准备播放: [{start}] 到 [{end}]，共 {end-start+1} 首")
        print(f"🔄 播放模式: {self._mode_name(mode)}")
        
        self.play_mode = mode
        self.range_start = start
        self.range_end = end
        self.current_index = start
        
        self._play_in_range()
    
    def _play_in_range(self):
        """在区间内播放"""
        if self.current_index > self.range_end:
            if self.play_mode == "loop":
                self.current_index = self.range_start
            else:
                print("\n✅ 区间播放完成")
                return
        
        self.play_by_index(self.current_index)
        
        # 监听播放结束，自动下一首（简化版，实际可能需要轮询）
        if self.play_mode != "single":
            # 这里简化处理，实际应用中可能需要检测音频播放状态
            pass
    
    def set_mode(self, mode):
        """设置播放模式"""
        modes = {
            '1': ('sequence', '顺序播放'),
            '2': ('random', '随机播放'),
            '3': ('single', '单曲循环'),
            '4': ('loop', '列表循环')
        }
        
        if mode in modes:
            self.play_mode = modes[mode][0]
            print(f"🔄 播放模式: {modes[mode][1]}")
        else:
            print("❌ 无效模式")
    
    def _mode_name(self, mode):
        """获取模式中文名"""
        names = {
            'sequence': '顺序播放',
            'random': '随机播放', 
            'single': '单曲循环',
            'loop': '列表循环'
        }
        return names.get(mode, mode)
    
    def add_favorite(self, index=None):
        """收藏歌曲"""
        if index is None:
            index = self.current_index
        
        self.favorites.add(index)
        song = self.playlist[index]
        print(f"♥ 已收藏: {song['artist']} - {song['title']}")
    
    def show_favorites(self):
        """显示收藏"""
        if not self.favorites:
            print("暂无收藏")
            return
        
        print(f"\n♥ 我的收藏 ({len(self.favorites)}首):")
        for idx in sorted(self.favorites):
            song = self.playlist[idx]
            print(f"  [{idx}] {song['artist']} - {song['title']}")
    
    def show_history(self):
        """显示播放历史"""
        if not self.play_history:
            print("暂无播放历史")
            return
        
        print(f"\n📜 最近播放 ({len(self.play_history)}首):")
        # 去重并保留顺序
        seen = set()
        unique_history = []
        for idx in reversed(self.play_history):
            if idx not in seen:
                seen.add(idx)
                unique_history.append(idx)
                if len(unique_history) >= 10:  # 显示最近10首
                    break
        
        for idx in unique_history:
            song = self.playlist[idx]
            print(f"  [{idx}] {song['artist']} - {song['title']}")
    
    def show_status(self):
        """显示当前播放状态"""
        if not self.is_playing:
            print("⏹️  停止状态")
            return
        
        song = self.playlist[self.current_index]
        status = "⏸️ 暂停" if self.is_paused else "▶️ 播放中"
        print(f"\n{status}: [{self.current_index}] {song['artist']} - {song['title']}")
        print(f"🔄 模式: {self._mode_name(self.play_mode)} | 卷: {song['vol']}")
    
    def interactive(self):
        """交互式终端"""
        print(f"""
{'='*80}
🎵 欢迎使用终端音乐播放器
{'='*80}
命令列表:
  play <索引>          - 播放指定歌曲 (如: play 0)
  range <起始> <结束>  - 播放区间 (如: range 10 20)
  next/n               - 下一首
  prev/p               - 上一首  
  pause/pp             - 暂停/继续
  stop/s               - 停止
  mode <模式>          - 设置模式 (1顺序 2随机 3单曲循环 4列表循环)
  search <关键词>      - 搜索歌曲
  list/ls              - 显示完整歌单
  fav                  - 收藏当前歌曲
  favs                 - 显示收藏
  history/his          - 播放历史
  status/st            - 播放状态
  help/?               - 显示帮助
  quit/q               - 退出
{'='*80}
""")
        
        while True:
            try:
                cmd = input("\n🎵 > ").strip().lower()
                if not cmd:
                    continue
                
                parts = cmd.split()
                action = parts[0]
                
                if action in ['quit', 'q', 'exit']:
                    self.stop()
                    print("👋 再见！")
                    break
                
                elif action == 'play':
                    if len(parts) > 1:
                        self.play_by_index(int(parts[1]))
                    else:
                        print("用法: play <索引>")
                
                elif action == 'range':
                    if len(parts) >= 3:
                        self.play_range(int(parts[1]), int(parts[2]), self.play_mode)
                    else:
                        print("用法: range <起始索引> <结束索引>")
                
                elif action in ['next', 'n']:
                    self.next_song()
                
                elif action in ['prev', 'p']:
                    self.prev_song()
                
                elif action in ['pause', 'pp']:
                    self.pause()
                
                elif action in ['stop', 's']:
                    self.stop()
                    print("⏹️  已停止")
                
                elif action == 'mode':
                    if len(parts) > 1:
                        self.set_mode(parts[1])
                    else:
                        print("用法: mode <1/2/3/4>")
                        print("  1: 顺序播放  2: 随机播放  3: 单曲循环  4: 列表循环")
                
                elif action == 'search':
                    if len(parts) > 1:
                        self.search(' '.join(parts[1:]))
                    else:
                        print("用法: search <关键词>")
                
                elif action in ['list', 'ls']:
                    self.show_library()
                
                elif action == 'fav':
                    self.add_favorite()
                
                elif action == 'favs':
                    self.show_favorites()
                
                elif action in ['history', 'his']:
                    self.show_history()
                
                elif action in ['status', 'st']:
                    self.show_status()
                
                elif action in ['help', '?', 'h']:
                    self.interactive()
                    return
                
                else:
                    print(f"❓ 未知命令: {action}，输入 help 查看帮助")
                    
            except KeyboardInterrupt:
                print("\n👋 再见！")
                self.stop()
                break
            except Exception as e:
                print(f"❌ 错误: {e}")


if __name__ == "__main__":
    player = TerminalMusicPlayer()
    player.interactive()



python music_terminal.py

# 终端交互示例：
🎵 > search 李荣浩          # 搜索
🎵 > play 25                # 播放第25首
🎵 > range 20 30            # 播放20-30首
🎵 > mode 2                 # 随机播放
🎵 > next                   # 下一首
🎵 > fav                    # 收藏当前歌曲
🎵 > favs                   # 查看收藏
🎵 > pause                  # 暂停
🎵 > quit                   # 退出


有 55 首歌曲，开始分卷（每卷<100MB）...

============================================================
尝试每卷 15 首...
============================================================

============================================================
构建第 1 卷 (15首歌)...
============================================================
  编码: AGA - 圆.mp3
  编码: EXO-M - History (Chinese Ver.).mp3
  编码: G.E.M.邓紫棋 - 透明.mp3
  编码: mj apanay,aren park - time machine (feat. aren park).mp3
  编码: Taylor Swift - You Belong With Me.mp3
  编码: TC - 熄灭.mp3
  编码: 万妮达Vinida Weng - 算了.mp3
  编码: 卓文萱 - 读心术.mp3
  编码: 卢广仲 - 狂迪.mp3
  编码: 周品 - 夸张.mp3
  编码: 周菲戈 - 拆穿 (Live).mp3
  编码: 孙燕姿 - 当冬夜渐暖.mp3
  编码: 孙燕姿 - 我想.mp3
  编码: 孙燕姿 - 爱情证书.mp3
  编码: 孙燕姿 - 绿光.mp3
  确认: 生成了 15 个 track 文件
  ✅ 包名已修改为: my-music-box-vol1
  构建 wheel...
  构建输出: ng 'my_music_box_vol1-1.0.1.dist-info/top_level.txt'
adding 'my_music_box_vol1-1.0.1.dist-info/RECORD'
removing build\bdist.win-amd64\wheel
Successfully built my_music_box_vol1-1.0.1-py3-none-any.whl

  ✅ my-music-box-vol1 完成！大小: 55.7MB

============================================================
构建第 2 卷 (15首歌)...
============================================================
  清理: build
  清理: my_music_box/encoded_tracks
  清理: my_music_box_vol1.egg-info
  编码: 孙燕姿 - 雨天.mp3
  编码: 孙燕姿 - 风衣.mp3
  编码: 张叶蕾 - 心跳.mp3
  编码: 张小只ya - 小模样.mp3
  编码: 张惠妹 - 人质.mp3
  编码: 张惠妹 - 听海.mp3
  编码: 张杰 - 千万次想象.mp3
  编码: 张靓颖,李秉成 - 终于等到你.mp3
  编码: 方大同 - Ring Finger.mp3
  编码: 方大同 - 才二十三.mp3
  编码: 李嘉格 - 沦陷2025.mp3
  编码: 李宗盛 - 晚婚 (Live).mp3
  编码: 李荣浩 - 太坦白.mp3
  编码: 李荣浩 - 恋人.mp3
  编码: 李荣浩 - 海陆风.mp3
  确认: 生成了 15 个 track 文件
  ✅ 包名已修改为: my-music-box-vol2
  构建 wheel...
  构建输出: ng 'my_music_box_vol2-1.0.2.dist-info/top_level.txt'
adding 'my_music_box_vol2-1.0.2.dist-info/RECORD'
removing build\bdist.win-amd64\wheel
Successfully built my_music_box_vol2-1.0.2-py3-none-any.whl

  ✅ my-music-box-vol2 完成！大小: 61.0MB

============================================================
构建第 3 卷 (15首歌)...
============================================================
  清理: build
  清理: my_music_box/encoded_tracks
  清理: my_music_box_vol2.egg-info
  编码: 李荣浩 - 笑忘书 (Live).mp3
  编码: 李荣浩 - 走走.mp3
  编码: 林俊杰 - 不潮不用花钱.mp3
  编码: 林俊杰 - 爱的鼓励.mp3
  编码: 林倛玉 - 一点一滴 (青蛙王子 主题曲).mp3
  编码: 林倛玉 - 同花顺.mp3
  编码: 梁博 - 日落大道 (Live版).mp3
  编码: 王菲 - 归途有风.mp3
  编码: 王菲 - 清风徐来.mp3
  编码: 胡彦斌 - 味道 (Live).mp3
  编码: 茜拉 - 到时说爱我.mp3
  编码: 萧敬腾 - Say A Lil Something.mp3
  编码: 蔡依林 - 看我72变.mp3
  编码: 蔡健雅 - 舞步.mp3
  编码: 薛凯琪 - 蘇州河 (慕容雪 - Mandarin Version).mp3
  确认: 生成了 15 个 track 文件
  ✅ 包名已修改为: my-music-box-vol3
  构建 wheel...
  构建输出: ng 'my_music_box_vol3-1.0.3.dist-info/top_level.txt'
adding 'my_music_box_vol3-1.0.3.dist-info/RECORD'
removing build\bdist.win-amd64\wheel
Successfully built my_music_box_vol3-1.0.3-py3-none-any.whl

  ✅ my-music-box-vol3 完成！大小: 61.5MB

============================================================
构建第 4 卷 (10首歌)...
============================================================
  清理: build
  清理: my_music_box/encoded_tracks
  清理: my_music_box_vol3.egg-info
  编码: 袁娅维TIA RAY,黄丽玲,万妮达Vinida Weng - 惊奇无限假期 A-T-V.mp3
  编码: 许嵩 - 燕归巢.mp3
  编码: 郭采洁 - 该忘了.mp3
  编码: 金海心 - 阳光下的星星.mp3
  编码: 铃凯,吴青峰 - 一个人.mp3
  编码: 陈奕迅 - 内疚.mp3
  编码: 陈粒 - 清透.mp3
  编码: 陈粒 - 空空.mp3
  编码: 陶喆 - 蝴蝶.mp3
  编码: 韦礼安 - Luvin' U.mp3
  确认: 生成了 10 个 track 文件
  ✅ 包名已修改为: my-music-box-vol4
  构建 wheel...
  构建输出: ng 'my_music_box_vol4-1.0.4.dist-info/top_level.txt'
adding 'my_music_box_vol4-1.0.4.dist-info/RECORD'
removing build\bdist.win-amd64\wheel
Successfully built my_music_box_vol4-1.0.4-py3-none-any.whl

  ✅ my-music-box-vol4 完成！大小: 40.6MB

============================================================
✅ 分卷构建成功！
============================================================
  Vol1: 15首, 55.7MB
  Vol2: 15首, 61.0MB
  Vol3: 15首, 61.5MB
  Vol4: 10首, 40.6MB

共 4 卷
