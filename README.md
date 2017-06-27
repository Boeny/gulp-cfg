# gulp-generator
Генерирует gulpfile.js, упрощая его написание, догружает необходимые плагины и убирает неактуальные из package.json.

Принимает в качестве аргумента объект, содержащий задания (таски), порядок объявления не важен.
Именем таска является ключ объекта:

	var util = require('gulp-util');

	require('gulp-generator')({
		clean: {// == name
			src: ['dist', {read: false}],// <=> {dist: {read: false}}
			pipe: 'clean'
		},
		
		// или:
		clean: {
			pipe: [['src', 'dist', {read: false}], 'clean']// <=> {src: ['dist', {read: false}]}
		},

		minify: {
			src: 'src/index.js',// <=> ['src/index.js'],
			pipe: [
				{concat: 'vendor.js'},
				'sourcemaps.init',
				'filesize','babel','filesize',
				{dest: 'dist/bundle.js'},
				'uglify','filesize',
				['sourcemaps.write', '.'],
				{rename: 'vendor.min.js'},
				{dest: 'dist'},
				{error: util.log}
			]
		},
		
		// или:
		minify: {
			pipe: [['src', 'dist'], ...]// <=> {src: 'dist'}
		},
		
		// или:
		minify: [...]
	});

# Потоки
Зарезервированные ключи таска: __src, dest, error, on, watch, done, \_return, depends, tasks__

Элементы потока это плагины, вызываемые в pipe(plugin()):
- объект с одним полем (ключ - название плагина __без "gulp-"__, значение - аргументы)
- массив (первый элемент - название плагина, остальные - аргументы)
- строка (название плагина, вызывается без аргументов)

Тело таска может быть массивом описания потока:

	require('gulp-generator')({
		clean: [['src', 'dist', {read: false}], 'clean']
	});

или объектом (отсутствие поля __pipe__ допустимо, если плагин один)

	require('gulp-generator')({
		clean: {
			src: ['dist', {read: false}],
			clean: ''// clean() без аргументов
		}
	});

Тогда ключ - название плагина, значение - его аргументы.
Если плагин содержит точку, то вызывается метод плагина.

Если в качестве аргументов задана функция, то она вычисляет аргумент:

	require('gulp-generator')({
		css: {
			_return: true,
			src: 'src/style.less',
			pipe: [
				{changed: 'build/css'},
				
				['less', () => {// вычисляем аргумент
					var autoprefixer = require('less-plugin-autoprefix');
					return {
						plugins: [new autoprefixer({browsers: ['last 2 versions']})]
					};
				}],
				
				{dest: 'dist/main.css'},
				{error: util.log}
			]
		}
	});

# Поле \_return
Если нужно передать результат обработки от одного таска другому, нужно использовать поле __\_return__:

	require('gulp-generator')({ default: {
		_return: true,
		src: './src/index.js',
		pipe: [
			'babel', 'uglify',
			{dest: 'dist/bundle.js'}
		]
	}});

Будет преобразовано в:

	const babel = require('gulp-babel');
	const uglify = require('gulp-uglify');
	
	gulp.task('default', function(){
		return gulp.src('./src/index.js')
			.pipe(babel())
			.pipe(uglify())
			.pipe(gulp.dest('dist/bundle.js'));
	});
	

# task как массив потоков
Таск может быть массивом потоков, если в массиве хотя бы один объект содержит больше одного ключа, причем поле __\_return__ может быть только у последнего:

	require('gulp-generator')({
		default: [
			{src: './src/*.js', jsdoc: './documentation'},
			{
				_return: true,
				src: './src/*.js',
				pipe: [
					'soucemaps.init', 'babel',
					{concat: 'bundle.js'},
					{'sourcemaps.write': '.'},
					{dest: 'dist'}
				]
			}
		]
	});

Таск также может содержать массив потоков в поле __tasks__, если нужно ввести общие для них поля (объекты могут содержать один ключ):

	require('gulp-generator')({
		default: {
			done: true,
			src: './src/*.js',
			tasks: [
				{jsdoc: './documentation'},
				{
					_return: true,// игнорируется из-за done
					pipe: [
						'soucemaps.init', 'babel',
						{concat: 'bundle.js'},
						{'sourcemaps.write': '.'},
						{dest: 'dist'}
					]
				}
			]
		}
	});

# task как функция
Тело таска может быть функцией

	require('gulp-generator')({
		default: function(gulp, done){
			gulp.src('src/index.js')
				.pipe(require('gulp-uglify')())
				.pipe(gulp.dest('dist/bundle.js'));
			
			gulp.watch('less/**/*.less');

			done();
		}
	});

# Поле depends
Ключ __depends__ в виде строки или массива строк определяет, завершение каких задач следует дождаться:

	require('gulp-generator')({
		'buildTests':{
			depends: 'default',
			_return: true,
			pipe: [{src: './tests/*.js'}, 'babel', {dest: 'built-tests'}]
		},

		'test':{
			depends: 'buildTests',
			src: ['./built-tests/test.js', {read:false}],
			pipe: {mocha: {reporter:'nyan'}}
		}
	});
	
Будет преобразовано в:
	
	const soucemaps = require('gulp-sourcemaps');
	const babel = require('gulp-babel');
	const mocha = require('gulp-mocha');
	
	gulp.task('buildTests', ['default'], function(){
		return gulp.src('./tests/*.js')
			.pipe(babel())
			.pipe(gulp.dest('built-tests'));
	});
	gulp.task('test', ['buildTests'], function(){
		gulp.src('./built-tests/test.js', {read:false})
			.pipe(mocha({reporter:'nyan'}));
	});

# Поле done
Если в теле таска поле __done == true__, то после всех потоков будет вставлена функция __done()__ (поле __\_return__ игнорируется)

	require('gulp-generator')({
		default: {
			done: true,
			_return: true,// игнорируется
			src: './src/*.js',
			pipe: [
				'soucemaps.init',
				'babel',
				{'concat': 'bundle.js'},
				{'sourcemaps.write': '.'},
				['dest': 'dist'}
			]
		}
	});

Будет преобразовано в:

	const soucemaps = require('gulp-sourcemaps');
	const babel = require('gulp-babel');
	const concat = require('gulp-concat');
	
	gulp.task('default', function(done){
		gulp.src('./src/*.js')
			.pipe(soucemaps.init())
			.pipe(babel())
			.pipe(concat('bundle.js'))
			.pipe(sourcemaps.write('.'))
			.pipe(gulp.dest('dist'));

		done();
	});

# Поле watch
Ключ __watch__ таска является методом объекта __gulp__ и принимает в себя аргумент или массив аргументов из одного или двух элементов.

Если второй аргумент - объект с ключом __change__, то вызывается __watch(first_argument).on('change', callback)__.

Значением ключа __change__ должна быть функция, возвращающая объект/массив описания потока:

	require('gulp-generator')({
		'tdd-single': {
			_return: true,
			watch: ['src/**/*.spec.js', {change: (file) => {
				return {src: file.path, mocha: {compilers: 'babel'}};
			}]
		},
		
		// или:
		'tdd-single': {watch: 'src/**/*.spec.js'}
	});

В ином случае вторым аргументом должен быть объект/массив, описывающий поток, который будет вызываться в __watch(first_argument, (files) => files.pipe...)__

	const util = require('gulp-util');
	
	require('gulp-generator')({
		'css:watch':{
			watch: [
				{
					glob: 'less/**/*.less',
					emit: 'one',
					emitOnGlob: false
				},
				{
					less:'',
					dest:'dist/main.css',
					error: util.log
				}
			]
		}
	});
