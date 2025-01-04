--- 
layout: single
classes: wide
title: "[AI/ML] DL4J Overview"
header:
  overlay_image: /img/ai-ml-bg.png
excerpt: 'JVM 기반으로 분산 딥러닝 학습이 가능하고 다른 런타임과의 상호 운용성 및 대규모 데이터 처리가 가능한 DL4J 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - AI/ML
tags:
    - Practice
    - ML
    - DL4J
    - Java
    - DL
    - DeepLearning
toc: true
use_math: true
---

## DL4J
`Eclipse Deeplearning4j(DL4J`)는 `Java Virtual Machine(JVM)` 을 위한 오픈소스 분산 딥러닝 라이브러리이다. 
주로 `Java` 로 작성되었고 성능 최적화 를 위해 일부 네이티브 `C++` 컴포넌트를 포함하고 있다. 
`Java` 에서 모데를 학습시키는 동시에 `Python` 상태계와 상호작용할 수 있는 유일한 프레임워크로, 
`cpython` 바인딩을 통한 `Python` 실행 및 모델 가져오기 `tensorflow-java` 과 같은 다른 런타임과의 상호 운용성을 제공한다.  

또한 `Java` 나 `Scala` 를 사용해 분산 학습을 지원하고 `Apache Hadoop` 과 `Spark` 와 같은 빅데이터 프레임워크와 원할하게 통합되며, 
대규모 엔터프라이즈급 솔루션에 적합하다. 
이러한 `DL4J` 의 이점을 간략하게 정리하면 아래와 같다.  

- `JVM` 기반 : `Java` 및 `Scala` 로 작성되어 기존 `JVM` 애플리케이션과 쉽게 통합 가능하다. 
- `분산 학습` : 다수의 `GPU` 및 클러스터를 통한 대규모 학습을 지원한다. 
- `다양한 신경망 지원` : 피드포워드(`feed forward`), 순환 신경망(`RNN`), 합성곱 신경망(`CNN`) 등 다양한 신경망 아키텍쳐를 지원한다. 
- `빅데이터 도구와의 통합` : `Hadoop`, `Spark` 및 `Kubernetes` 와 함께 통합해 대규모 데이터 처리를 구현할 수 있다. 
- `맞춤화 가능` : 네트워크 구성 및 하이퍼파라미터 튜닝을 위한 유연성을 제공한다. 
- `호환성` : `Tensorflow`, `Keras` 와 같은 `Python` 라이브러리와의 `Import/Export` 기능으로 상호운용성을 제공한다. 


### Submodules
`DL4J` 는 다음과 같은 주요 구성 요소를 통해 구성된다.  

- `SameDiff` : 복잡한 그래프 실행을 위한 `Tensorflow/PyTorch` 와 유사한 프레임워크이다. 저수준이지만 유연하고, `onnx` 및 `Tensorflow` 그래프 실행을 위한 기본 `API` 이다. 
- `ND4J` : `Java` 용 `Numpy++` 로, `Numpy` 연산과 `Tensorflow/PyTorch` 연산의 조합을 포함한다. 
- `LibND4j` : 다양한 장치에서 수학 코드를 실행할 수 있도록 설계된 경량 독립형 `C++` 라이브러리다. 다양한 장치에서 실행 가능하도록 최적화할 수 있다. 
- `Python4J` : `Python` 스크립트의 프로덕션 배포를 간소화하는 `Python` 스크립트 실행 프레잌워크이다. 
- `Apache Spark Integration` : `Apache Spark` 프레이뭐크와 통합하여 `Spark` 에서 딥러닝 파이프라인 실행을 지원한다. 
- `DataVec` : 원시 입력 데이터를 신경망 실행에 적합한 텐서로 변환하는 데이터 변환 라이브러리이다. ETL(추출, 변환, 로드) 작업을 처리해 데이터 정규화, 특징 엔지니어링, 구조화된 데이터와 비구조화된 데이터를 처리할 수 있다. 
- `Model Zoo` : 이미지 인식, 자연어 처리(`NLP`)와 같은 일반적인 작업을 위한 사전 학습된 모델을 제공하여 모델 개발 시간을 절약할 수 있다. 


### Demo
`DL4J` 를 사용해서 간단한 모델 기반 학습을 한 뒤 `input` 에 따른 예측결과 `output` 을 도출하는 간단한 예제에 대해 살펴본다. 
데모의 자세한 코드 내용은 [여기](https://github.com/windowforsun/dl4j-simple-demo)
에서 확인할 수 있다.  

우선 `DL4J` 사용에 필요한 아래와 같은 의존성을 추가한다.  

```groovy
ext {
    dl4jVersion = '1.0.0-M2.1'
}

dependencies {
    // only nd4j for example
    implementation "org.nd4j:nd4j-native:${dl4jVersion}"
    // apple silicon
    implementation "org.nd4j:nd4j-native:${dl4jVersion}:macosx-arm64"
    implementation "org.nd4j:nd4j-native-platform:${dl4jVersion}"
    implementation "org.deeplearning4j:deeplearning4j-datasets:${dl4jVersion}"
    implementation "org.deeplearning4j:deeplearning4j-core:${dl4jVersion}"

    // + dl4j for example
    implementation "org.datavec:datavec-api:${dl4jVersion}"
    implementation "org.datavec:datavec-data-image:${dl4jVersion}"
    implementation "org.datavec:datavec-local:${dl4jVersion}"
    implementation "org.deeplearning4j:resources:${dl4jVersion}"
    implementation "org.deeplearning4j:deeplearning4j-ui:${dl4jVersion}"
    implementation "org.deeplearning4j:deeplearning4j-zoo:${dl4jVersion}"
    implementation "org.deeplearning4j:deeplearning4j-parallel-wrapper:${dl4jVersion}"
}
```  

첫 번째 예제는 `XOR` 연산의 `input` 과 `output` 을 학습 시킨 모델을 만들어 `input` 을 넣었을 때 `XOR` 연산의 결과를 에측하는 예제이다.  

```java
public class XorExample {

	public static void main(String[] args) {
		// 1. 신경망 구성 정의
		MultiLayerConfiguration conf = new NeuralNetConfiguration.Builder()
			.updater(new Sgd(0.1))
			.list()
			.layer(new DenseLayer.Builder()
				.nIn(2) // 입력 노드 개수
				.nOut(3) // 은닉층 노드 개수
				.activation(Activation.RELU) // 활성화 함수
				.build())
			.layer(new OutputLayer.Builder(LossFunctions.LossFunction.MSE) // 출력층
				.nIn(3) // 은닉층 노드 개수
				.nOut(1) // 출력 노드 개수
				.activation(Activation.IDENTITY) // 활성화 함수
				.build())
			.build();

		// 2. 모델 초기화
		MultiLayerNetwork model = new MultiLayerNetwork(conf);
		model.init();

		// 3. 학습 데이터 생성 (XOR 문제)
		double[][] input = {
			{0, 0},
			{0, 1},
			{1, 0},
			{1, 1}
		};
		double[][] labels = {
			{0},
			{1},
			{1},
			{0}
		};

		DataSet dataset = new DataSet(Nd4j.create(input), Nd4j.create(labels));

		// 4. 모델 학습
		for (int i = 0; i < 1000; i++) { // 1000번의 에포크 학습
			model.fit(dataset);
		}

		// 5. 예측 결과 출력
		System.out.println("예측 결과:");
		for (double[] sample : input) {
			double[] prediction = model.output(Nd4j.create(new double[][] {sample})).toDoubleVector();
			System.out.println(Nd4j.create(sample) + " -> " + Nd4j.create(prediction));
		}
	}
}
```  

코드를 실행하면 아래와 같이 `XOR` 의 두 `input` 이 주어졌을 때 대략적인(소수점 단위 포함) 연산의 예측결과를 도출해 내는 것을 확인할 수 있다.  

```bash
예측 결과:
[         0,         0] -> [4.9219e-7]
[         0,    1.0000] -> [1.0000]
[    1.0000,         0] -> [1.0000]
[    1.0000,    1.0000] -> [4.9184e-7]
```  

다음은 `LinearRegression(선형 회귀)` 사용해서 `y = 2x + 1` 수식에 대해서 `x` 가 입력일 때 `y` 의 결과를 예측하는 모델을 학습시키는 예제를 살펴본다.  

```java
public class LinearRegressionExample {

	public static void main(String[] args) {
		// 1. 신경망 구성 정의
		MultiLayerConfiguration conf = new NeuralNetConfiguration.Builder()
			.updater(new Sgd(0.001)) // 학습률
			.list()
			.layer(new DenseLayer.Builder()
				.nIn(1) // 입력 노드 개수 (x)
				.nOut(10) // 은닉층 노드 개수
				.activation(Activation.RELU) // 활성화 함수
				.build())
			.layer(new OutputLayer.Builder(LossFunctions.LossFunction.MSE) // 출력층
				.nIn(10) // 은닉층 노드 개수
				.nOut(1) // 출력 노드 개수 (y)
				.activation(Activation.IDENTITY) // 선형 활성화 함수
				.build())
			.build();

		// 2. 모델 초기화
		MultiLayerNetwork model = new MultiLayerNetwork(conf);
		model.init();

		// 3. 학습 데이터 생성
		int numSamples = 100; // 데이터 샘플 수
		double[][] input = new double[numSamples][1]; // 입력 데이터
		double[][] labels = new double[numSamples][1]; // 출력 데이터
		for (int i = 0; i < numSamples; i++) {
			double x = i * 0.1; // 입력값 생성
			// double x = i; // 입력값 생성
			input[i][0] = x;
			labels[i][0] = 2 * x + 1; // y = 2 * x + 1
		}
		DataSet dataset = new DataSet(Nd4j.create(input), Nd4j.create(labels));

		// 4. 모델 학습
		int numEpochs = 1000; // 학습 에포크 수
		for (int i = 0; i < numEpochs; i++) {
			model.fit(dataset);
		}

		// 5. 예측 결과 출력
		System.out.println("예측 결과:");
		double[][] testInputs = {{0d}, {1d}, {2d}, {3d}, {4d}}; // 테스트 입력값
		for (double[] testInput : testInputs) {
			double[] prediction = model.output(Nd4j.create(new double[][] {testInput})).toDoubleVector();
			System.out.printf("입력: %.1f -> 예측 출력: %.3f\n", testInput[0], prediction[0]);
		}
	}
}
```  

실행하면 아래와 같이, 정확한 결과는 아니지만 소수점 단위의 오차정도로 `y = 2x + 1` 에서 `y` 결과를 예측하는 것을 확인할 수 있다.  

```
예측 결과:
입력: 0.0 -> 예측 출력: 0.865
입력: 1.0 -> 예측 출력: 2.886
입력: 2.0 -> 예측 출력: 4.906
입력: 3.0 -> 예측 출력: 6.927
입력: 4.0 -> 예측 출력: 8.947
```  


---  
## Reference
[Deeplearning4j Suite Overview](https://deeplearning4j.konduit.ai/)  

