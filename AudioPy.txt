
import os
import time,datetime
import threading
from tkinter import *
from tkinter import (messagebox as msg,filedialog as fd, ttk)
from ttkthemes import themed_tk as tk
from scipy.io import wavfile as wav
import numpy as np

from pygame import mixer
##from pydub import AudioSegment 
import librosa as lb
from mutagen.mp3 import MP3
from matplotlib import pyplot as plt,ticker
from matplotlib.backend_bases import key_press_handler
from matplotlib.figure import Figure
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg

class Window(Frame):
    
    global filename_path   #global variable contains opening file location(path)
    def __init__(self, master=None):    #self- object for class 'window'
        Frame.__init__(self, master)
        mixer.init()        #mixer method from pygame module is initialized
        self.master = master
        self.init_window()

    #Creation of init_window
    def init_window(self):    
        self.master.title("Audio Processor")
        self.pack(fill=BOTH, expand=1)
        menu = Menu(self.master)         #menu bar for tkinter gui is initialized
        self.master.config(menu=menu) 

        self.playlist=[]   #class list variable 'playlist' is declared for storing list of loading audio files 
        self.flag=0     #flag for opening visualizer
        self.ctrl=1
        self.cutflag=0   #flag for opening cut submenu
        self.cutterflag=0
        self.speedflag=0  #flag for opening speed submenu
        
        file = Menu(menu)
        file.add_command(label="Open",command=self.browse_file)
        file.add_command(label="SaveAs",command=self.save)
        file.add_command(label="Exit", command=self.client_exit)
        menu.add_cascade(label="File", menu=file) #adds "file" to our menu
        
        tools=Menu(menu)
        tools.add_command(label="Cut",command=self.cut)
        tools.add_command(label="Speed",command=self.speed)
        menu.add_cascade(label="Tools",menu=tools) #adds "Tools" to our menu
        
        help1=Menu(menu)        
        help1.add_command(label="About Us", command=self.about_us)
        menu.add_cascade(label="Help", menu=help1)  #adds "Help" to our menu

        leftframe = Frame(self)      #separation of window frame into various frames
        leftframe.pack(side=LEFT)    #sets the frame into gui window

        topframe1=Frame(leftframe)
        topframe1.pack()

        self.playlistbox = Listbox(topframe1,selectbackground='blue',bd=5)
        self.playlistbox.pack()      #creation of playlist box
        
        addBtn = ttk.Button(topframe1, text="+ Add", command=self.browse_file)
        addBtn.pack(side=LEFT)       #add button is added up

        def del_song(): #function to delete a file from playlist box
            selected_song = self.playlistbox.curselection()
            selected_song = int(selected_song[0])
            self.playlistbox.delete(selected_song)
            self.playlist.pop(selected_song)

        delBtn = ttk.Button(topframe1, text="- Del", command=del_song)
        delBtn.pack(side=LEFT)        
        
        topframe11=Frame(leftframe)
        topframe11.pack()

        visual=ttk.Label(topframe11,text='VISUALIZER :')
        visual.pack()

        okbtn=ttk.Button(topframe11,text="SELECT",command=self.histo)
        okbtn.pack(side=LEFT)

        chgbtn=ttk.Button(topframe11,text="CHANGE",command=self.change)
        chgbtn.pack(side=LEFT)


        topframe = Frame(leftframe)   #creates subframe under leftframe for playing buttons
        topframe.pack(pady=10)

        self.lengthlabel = ttk.Label(topframe, text='Total Length : --:--')
        self.lengthlabel.pack(pady=5)

        self.currenttimelabel = ttk.Label(topframe, text='Current Time : --:--', relief=GROOVE)
        self.currenttimelabel.pack()


        def play_music():      #function for playing file
            if self.paused:
                mixer.music.unpause()
                statusbar['text'] = "Music Resumed"
                self.paused = FALSE
            else:
                try:
                    if self.cutflag==0:
                        self.stop_music()       
                        time.sleep(1)
                        selected_song = self.playlistbox.curselection()  #selects file from playlist box   
                        selected_song = int(selected_song[0])
                        play_it = self.playlist[selected_song]
                        mixer.music.load(play_it)   
                    
                    mixer.music.play()
                    
                    if self.cutflag==0:
                        statusbar['text'] = "Playing music" + ' - ' + os.path.basename(play_it)
                        self.show_details(play_it)
                    else:
                        statusbar['text'] = "Playing Edited music"
                
                except:
                    msg.showerror('File not found', 'Melody could not find the file. Please check again.')


        self.paused = FALSE    #class variable to have control on pause music


        def pause_music():     
            self.paused = TRUE
            mixer.music.pause()
            statusbar['text'] = "Music paused"


        def rewind_music():
            play_music()
            statusbar['text'] = "Music Rewinded"


        def set_vol(val):
            volume = float(val) / 100
            mixer.music.set_volume(volume)
            # set_volume of mixer takes value only from 0 to 1. Example - 0, 0.1,0.55,0.54.0.99,1


        self.muted = FALSE


        def mute_music():
            if self.muted:  # Unmute the music
                mixer.music.set_volume(0.7)
                volumeBtn.configure(image=self.volumePhoto)
                scale.set(70)
                self.muted = FALSE
            else:  # mute the music
                mixer.music.set_volume(0)
                volumeBtn.configure(image=self.mutePhoto)
                scale.set(0)
                self.muted = TRUE


        middleframe = Frame(leftframe)
        middleframe.pack(pady=30)

        self.playPhoto = PhotoImage(file='play.png')  #displays image on the button
        playBtn = ttk.Button(middleframe, image=self.playPhoto,width=3, command=play_music)
        playBtn.grid(row=0, column=0, padx=10)

        self.stopPhoto = PhotoImage(file='stop.png')
        stopBtn = ttk.Button(middleframe, image=self.stopPhoto,width=3, command=self.stop_music)
        stopBtn.grid(row=0, column=2, padx=10)

        self.pausePhoto = PhotoImage(file='pause.png')
        pauseBtn = ttk.Button(middleframe, image=self.pausePhoto,width=3, command=pause_music)
        pauseBtn.grid(row=0, column=1, padx=10)

        # Bottom Frame for volume, rewind, mute etc.

        bottomframe = Frame(leftframe)
        bottomframe.pack()

        self.rewindPhoto = PhotoImage(file='rewind.png')
        rewindBtn = ttk.Button(bottomframe, image=self.rewindPhoto, width=3,command=rewind_music)
        rewindBtn.grid(row=0, column=0,padx=10)

        self.mutePhoto = PhotoImage(file='mute.png')
        self.volumePhoto = PhotoImage(file='volume.png')
        volumeBtn = ttk.Button(bottomframe, image=self.volumePhoto, command=mute_music)
        volumeBtn.grid(row=0, column=1)

        scale = ttk.Scale(bottomframe, from_=0, to=100, orient=HORIZONTAL, command=set_vol)
        scale.set(70)  # implement the default value of scale when music player starts
        mixer.music.set_volume(0.7)
        scale.grid(row=0, column=2, pady=15, padx=30)
        

    def show_details(self,play_song):   # gets the length of file
        file_data = os.path.splitext(play_song)

        if file_data[1] == '.mp3':
            audio = MP3(play_song)
            total_length = audio.info.length 
        else:
            a = mixer.Sound(play_song)
            total_length = a.get_length()

        # div - total_length/60, mod - total_length % 60
        mins, secs = divmod(total_length, 60)
        mins = round(mins)
        secs = round(secs)
        timeformat = '{:02d}:{:02d}'.format(mins, secs)
        self.lengthlabel['text'] = "Total Length" + ' - ' + timeformat

        t1 = threading.Thread(target=self.start_count, args=(total_length,))
        t1.start()
        

    def start_count(self, t):
            # mixer.music.get_busy(): - Returns FALSE when we press the stop button (music stop playing)
            # Continue - Ignores all of the statements below it. We check if music is self.paused or not.
        current_time = 0
        while current_time <= t and mixer.music.get_busy():
            if self.paused:
                continue
            else:
                mins, secs = divmod(current_time, 60)
                mins = round(mins)
                secs = round(secs)
                timeformat = '{:02d}:{:02d}'.format(mins, secs)
                self.currenttimelabel['text'] = "Current Time" + ' - ' + timeformat
                time.sleep(1)
                current_time += 1

        
    def stop_music(self):
        mixer.music.stop()
        statusbar['text'] = "Music Stopped"
            
    def about_us(self):
        msg.showinfo('About','Vaaimaiye vellum')

    def browse_file(self):    # function to open file from drive
        global filename_path
        filename_path = fd.askopenfilename(filetypes = (("MP3 Format Sound","*.mp3"),("wav files","*.wav"),("all files","*.*")))
        self.add_to_playlist(filename_path)        
        mixer.music.queue(filename_path)


    def add_to_playlist(self,filename):    # function to add file from drive to playlist box
        filename = os.path.basename(filename)
        index = 0
        self.playlistbox.insert(index, filename)
        self.playlist.insert(index, filename_path)
        time.sleep(1)
        index += 1

    def save(self):       #function to save worked file to drive
        files=[("wav files","*.wav"),("MP3 Format Sound","*.mp3"),("all files","*.*")]
        file=fd.asksaveasfilename(filetypes=files,defaultextension=".wav")
        lb.output.write_wav(file, self.aud, self.rate)

##    def cut_play(self):
##        lb.output.write_wav("temp.wav", self.aud, self.rate)
##        sound= AudioSegment.from_wav("temp.wav")
##        sound.export('temp.mp3',format='mp3')
##        mixer.music.stop()
##        mixer.music.queue("temp.mp3")
##        self.currenttimelabel['text'] = "Current Time" + ' - ' + "00:00"
##        self.show_details("temp.mp3")
        
        
    def cut(self):          #function definition enables cut audio file 
        self.cutterflag=1
        def cutter():
            start=self.txt1.get()
            end=self.txt2.get()
            dt1=datetime.datetime.strptime(start,"%M:%S")
            td1=dt1-datetime.datetime(1900,1,1)
            start=td1.total_seconds()
            dt2=datetime.datetime.strptime(end,"%M:%S")
            td2=dt2-datetime.datetime(1900,1,1)
            end=td2.total_seconds()
            
            end= int (end * self.rate)
            if end > len(self.aud):
                self.aud = self.aud[int( start* self.rate):]
            else:
                self.aud = self.aud[int( start* self.rate): end]
            self.cutflag=1
            self.ctrl=1
            self.flag=1
            self.histo()
            
        def remove():               # function disables opptions from the window
            self.cutButton.pack_forget()
            self.okButton.pack_forget()
            
        self.okButton = ttk.Button(self,text='Ok',command=remove)
        self.okButton.pack(side=RIGHT)
        self.cutButton = ttk.Button(self,text='Cut',command=cutter)
        self.cutButton.pack(side=RIGHT)

    

    def speed(self):     #function enables changing speed options
        self.slide.pack_forget()
        self.slide2.pack_forget()
        self.txt1.pack_forget()
        self.lbl1.pack_forget()
        self.txt2.pack_forget()
        self.lbl2.pack_forget()
        lbl3 = Label(self, text="Speed : ", anchor=W)
        lbl3.pack(side=LEFT)
        speed_rate=44100     #default frequency for audio files

        def set_speed(val):
            nonlocal speed_rate
            change_rate=(float(val)/10)
            speed_rate=int(self.rate* change_rate)
            txt3.delete(0,END)
            txt3.insert(0,change_rate)
            
            
        speed_scale = Scale(self, from_=0, to=20, orient=HORIZONTAL, showvalue=0, command=set_speed)
        speed_scale.set(10)        #scale to choose speed
        speed_scale.pack(side=LEFT)
        txt3=Entry(self,width=5,bd=5)
        txt3.pack(side=LEFT,padx=5)
        self.speedflag=1

        def remove():    #disables speed options
            lbl3.pack_forget()
            speed_scale.pack_forget()
            txt3.pack_forget()
            okButton.pack_forget()
            self.rate=speed_rate
        
        okButton = ttk.Button(self,text='Ok',command=remove)
        okButton.pack(side=RIGHT)
        
        
        
    def change(self):      #function to clear visualizer graph of old file
        res = msg.askyesno('Song Change','Do yo want to change song?')
        if res==True and self.ctrl==0:
            self.canvas.get_tk_widget().pack_forget()
            self.slide.pack_forget()
            self.slide2.pack_forget()
            self.txt1.pack_forget()
            self.lbl1.pack_forget()
            self.txt2.pack_forget()
            self.lbl2.pack_forget()
            if self.cutflag==1 or self.cutterflag==1:
                self.cutButton.pack_forget()
                self.okButton.pack_forget()

            if self.speedflag==1:
                lbl3.pack_forget()
                speed_scale.pack_forget()
                txt3.pack_forget()
                okButton.pack_forget()
            
            aud=np.zeros(10)
            time1 = np.arange(10)
            self.fig = Figure(figsize=(5,4), dpi=100)
            self.fig,ax1 = plt.subplots()
            cur_axes = plt.gca()
            formatter = ticker.FuncFormatter(lambda ms, x: time.strftime('%M:%S', time.gmtime(ms)))
            ax1.xaxis.set_major_formatter(formatter)
            cur_axes.axes.get_yaxis().set_visible(False)
            ax1.plot(time1,aud,linewidth=0.01,color='red')            
            self.canvas = FigureCanvasTkAgg(self.fig,self)  # A tk.DrawingArea.
            self.canvas.get_tk_widget().pack(side=TOP, fill=X, expand=0)
            statusbar['text'] = "Changing song in Visualizer"
            ti=len(time1)
            last=time1[ti-1]
            self.slide=Scale(self,from_=0,to=last,orient=HORIZONTAL)
            self.slide.pack(fill=X)
            self.slide2=Scale(self,from_=0,to=last,orient=HORIZONTAL,showvalue=0)
            self.slide2.pack(fill=X)
            self.flag=1
            self.ctrl=1
            self.speedflag=0
            self.cutflag=0


    def histo(self):     #function opens visualizer graph for selected audio file
        if self.ctrl==1:
            if self.cutflag==0:
                selected_song = self.playlistbox.curselection()
                selected_song = int(selected_song[0])
                file_data = os.path.splitext(self.playlist[selected_song])
                if file_data[1] == '.mp3':
                    self.aud,self.rate=lb.core.load(self.playlist[selected_song],sr=None)        
                else:
                    self.rate,self.aud= wav.read(self.playlist[selected_song])

                    
            if self.cutflag==1 or self.cutterflag==1:
                self.cutButton.pack_forget()
                self.okButton.pack_forget()
##                self.cut_play()
                
            if self.speedflag==1:
                lbl3.pack_forget()
                speed_scale.pack_forget()
                txt3.pack_forget()
                okButton.pack_forget()
                
            time1 = np.arange(0, float(self.aud.shape[0]), 1) / self.rate
            if self.flag==1:
                self.canvas.get_tk_widget().pack_forget()
                self.slide.pack_forget()
                self.slide2.pack_forget()
                self.txt1.pack_forget()
                self.lbl1.pack_forget()
                self.txt2.pack_forget()
                self.lbl2.pack_forget()
                self.flag=0
       
            self.fig = Figure(figsize=(5,4), dpi=100)
            self.fig,ax1 = plt.subplots()
            cur_axes = plt.gca()
            formatter = ticker.FuncFormatter(lambda ms, x: time.strftime('%M:%S', time.gmtime(ms)))  #converts seconds to MM:SS (time) format
            ax1.xaxis.set_major_formatter(formatter)
            cur_axes.axes.get_yaxis().set_visible(False)  #disables y axis from graph
            ax1.plot(time1,self.aud,linewidth=0.01,color='red')
            
            self.canvas = FigureCanvasTkAgg(self.fig,self)  # tk.DrawingArea.
            self.canvas.get_tk_widget().pack(side=TOP, fill=X, expand=0)
            statusbar['text'] = "Visualizer"
            ti=len(time1)
            last=time1[ti-1]
            def on_left(val_1):  #changes start time based on slider move
                self.txt1.delete(0,END)
                self.txt1.insert(0,time.strftime('%M:%S', time.gmtime(round(int(val_1),2))))

            def on_right(val_2):   #changes end time based on slider move
                self.txt2.delete(0,END)
                self.txt2.insert(0,time.strftime('%M:%S', time.gmtime(round(int(val_2),2))))
                
            self.slide=Scale(self,from_=0,to=last,orient=HORIZONTAL,showvalue=0,command=on_left,label="Start")
            self.slide.pack(fill=X)
            self.slide2=Scale(self,from_=0,to=last,orient=HORIZONTAL,showvalue=0,command=on_right,label="End")
            self.slide2.pack(fill=X)
            
            self.ctrl=0
            self.cutflag=0
            self.lims()
        

    def lims(self):
        
        self.lbl1 = Label(self, text="Start time:", anchor=W)
        self.lbl1.pack(side=LEFT)
        self.txt1 = Entry(self,width=5)
        self.txt1.pack(side=LEFT,padx=10)
        self.txt1.focus()
        self.lbl2 = Label(self, text="End time:")
        self.lbl2.pack(side=LEFT,padx=10)
        self.txt2 = Entry(self,width=5)
        self.txt2.pack(side=LEFT)
        def onclick(event):   #function works according to mouse button clicks
            if str(event.button)=='MouseButton.RIGHT':
                x2=event.xdata
                self.txt2.delete(0,END)
                self.txt2.insert(0,time.strftime('%M:%S', time.gmtime(round(x2,2))))
                self.slide2.set(x2)
            else:
                x1=event.xdata
                self.txt1.delete(0,END)
                self.txt1.insert(0,time.strftime('%M:%S', time.gmtime(round(x1,2))))
                self.slide.set(x1)
##            print("button=%d, x=%d, y=%d, xdata=%f, ydata=%f" %(event.button, event.x, event.y, event.xdata, event.ydata))
        cid = self.fig.canvas.mpl_connect('button_press_event', onclick)
        
         
    def client_exit(self): #function to exit from application
        res = msg.askyesno('Exit','Do yo want to exit?')
        if res==True:           
            self.destroy()
            exit()
        

root = tk.ThemedTk()   #initializes application gui
root.geometry("1080x720")
root.get_themes()                 # Returns a list of all themes that can be set
root.set_theme("radiance") 
statusbar = ttk.Label(root, text="Welcome", relief=SUNKEN, anchor=W, font='Times 12 italic')
statusbar.pack(side=BOTTOM, fill=X)  #sets statusbar at bottom of window
app = Window(root)      #calling class 'window'

def on_closing(): #function works on ckicking close(cross) symbol
    app.stop_music()
    root.destroy()


root.protocol("WM_DELETE_WINDOW", on_closing)
root.mainloop()   #terminates application gui


