[https://github.com/pvigier/dependency-graph](https://github.com/pvigier/dependency-graph) + [https://www.geeksforgeeks.org/detect-cycle-in-a-graph/](https://www.geeksforgeeks.org/detect-cycle-in-a-graph/) = <3

```py
import os
import re
import argparse
import codecs
from collections import defaultdict

class Graph():
	def __init__(self):
		self.graph = defaultdict(list)

	def add_edge(self,u,v):
		self.graph[u].append(v)

	def print_cycles_util(self, v, visited, recStack):
		visited.append(v)
		recStack.append(v)
		if (v in self.graph):
			for neighbour in self.graph[v]:
				if neighbour not in visited:
					self.print_cycles_util(neighbour, visited, recStack)
				elif neighbour in recStack and neighbour != v:
					print("---")
					print(neighbour)
					print(v)
		recStack.remove(v)

	def print_cycles(self):
		visited = []
		recStack = []
		for node in self.graph:
			if node not in visited:
				self.print_cycles_util(node,visited,recStack)

include_regex = re.compile('#include\s+["<](.*)[">]')
valid_extensions = ['.h', '.hpp']

def normalize(path):
	filename = os.path.basename(path)
	end = filename.rfind('.')
	end = end if end != -1 else len(filename)
	return filename[:end]

def get_extension(path):
	return path[path.rfind('.'):]

def find_all_files(path):
	files = []
	for entry in os.scandir(path):
		if entry.is_dir():
			files += find_all_files(entry.path)
		elif get_extension(entry.path) in valid_extensions:
			files.append(entry.path)
	return files

def find_includes(path):
	f = codecs.open(path, 'r', "utf-8", "ignore")
	code = f.read()
	f.close()
	return [normalize(include) for include in include_regex.findall(code)]

def create_graph(folder):
	includepairs = Graph()
	files = find_all_files(folder)
	folder_to_files = defaultdict(list)
	for path in files:
		folder_to_files[os.path.dirname(path)].append(path)
	for folder in folder_to_files:
		for path in folder_to_files[folder]:
			node = normalize(path)
			includes = find_includes(path)
			for include in includes:
				includepairs.add_edge(normalize(path), include)
	includepairs.print_cycles()

if __name__ == '__main__':
	parser = argparse.ArgumentParser()
	parser.add_argument('folder', help='Path to the folder to scan')
	args = parser.parse_args()
	create_graph(args.folder)
```