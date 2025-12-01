import os
from building import *

cwd     = GetCurrentDir()
CPPPATH = [cwd]
src     = Glob('*.c')

group = DefineGroup('drag-n-drop', src, depend = [''], CPPPATH = CPPPATH)

list = os.listdir(cwd)
for item in list:
    if os.path.isfile(os.path.join(cwd, item, 'SConscript')):
        group = group + SConscript(os.path.join(item, 'SConscript'))

Return('group')
