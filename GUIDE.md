# GUIDE

## 1. Tổng quan dự án
Dự án này là một bài lab Terraform về triển khai môi trường AI trên cloud. Mục tiêu là dựng một hạ tầng có thể chạy mô hình ngôn ngữ `google/gemma-4-E2B-it` bằng `vLLM`, sau đó mở API OpenAI-compatible để gọi từ bên ngoài.

Repo hiện có hai biến thể triển khai:
- `terraform/`: phiên bản AWS
- `terraform-gcp/`: phiên bản Google Cloud Platform

Cả hai biến thể đều dùng cùng một ý tưởng kiến trúc:
- tạo mạng private cho máy GPU
- cho máy GPU ra internet qua NAT để tải image và model
- khởi động container `vllm/vllm-openai`
- đặt Load Balancer ở phía public để nhận request HTTP

## 2. Kiến trúc chung
Luồng hoạt động của dự án:
1. Terraform tạo network, subnet, firewall/security group và load balancer.
2. Một máy chủ GPU/private VM được tạo ra trong mạng nội bộ.
3. Startup script cài Docker, cấu hình GPU runtime, rồi chạy container vLLM.
4. Load Balancer chuyển request vào cổng `8000` của vLLM.
5. Người dùng gọi API qua endpoint công khai để thử inference.

Mô hình mặc định được cấu hình trong repo là:
- `google/gemma-4-E2B-it`

## 3. Phiên bản AWS
Thư mục `terraform/` triển khai trên AWS với các thành phần chính:
- `VPC` với public subnet và private subnet
- `Internet Gateway` cho public subnet
- `NAT Gateway` để máy private đi ra internet
- `Bastion Host` ở public subnet để SSH an toàn vào máy GPU
- `GPU node` dùng AMI Deep Learning Base OSS Nvidia Driver GPU AMI
- `Application Load Balancer` để public API

### Các file quan trọng
- `terraform/main.tf`: khai báo toàn bộ hạ tầng
- `terraform/variables.tf`: biến cấu hình như `aws_region`, `hf_token`, `model_id`
- `terraform/outputs.tf`: xuất `bastion_public_ip`, `alb_dns_name`, `endpoint_url`, `gpu_private_ip`
- `terraform/user_data.sh`: script cài Docker và chạy vLLM

### Điểm đáng chú ý
- Máy GPU dùng loại `g4dn.xlarge`
- Model được tải từ Hugging Face nên cần `HF_TOKEN`
- API endpoint chạy qua ALB ở cổng `80`, rồi forward vào vLLM ở cổng `8000`

## 4. Phiên bản GCP
Thư mục `terraform-gcp/` triển khai trên Google Cloud với các thành phần chính:
- `VPC` và private subnet
- `Cloud Router` và `Cloud NAT`
- `Firewall rules` cho IAP SSH và health check của Load Balancer
- `GPU node` là Compute Engine VM gắn GPU T4
- `External HTTP Load Balancer` để public API

### Các file quan trọng
- `terraform-gcp/main.tf`: khai báo network, VM, NAT, load balancer
- `terraform-gcp/variables.tf`: biến như `project_id`, `region`, `zone`, `machine_type`, `gpu_type`, `gpu_count`, `hf_token`
- `terraform-gcp/outputs.tf`: xuất `load_balancer_ip`, `api_endpoint`, `gpu_node_name`, `iap_ssh_command`
- `terraform-gcp/user_data.sh`: script cài Docker, NVIDIA toolkit và chạy vLLM

### Điểm đáng chú ý
- Project ID là biến bắt buộc
- VM mặc định dùng `n1-standard-4` + `nvidia-tesla-t4`
- SSH debug được thiết kế qua IAP, không cần Bastion riêng
- API endpoint được public qua IP của Load Balancer

## 5. Startup script làm gì
Hai file `user_data.sh` đều có vai trò giống nhau: chuẩn bị máy chủ để chạy inference.

Trên AWS:
- bật Docker
- kéo image `vllm/vllm-openai:latest`
- chạy container với `HF_TOKEN` và `model_id`

Trên GCP:
- cài Docker và NVIDIA container toolkit
- cấu hình runtime cho Docker để dùng GPU
- chạy container vLLM với token Hugging Face

Kết quả cuối cùng là một OpenAI-compatible server trên cổng `8000`.

## 6. Cách chạy nhanh
### AWS
```bash
cd terraform
terraform init
export TF_VAR_hf_token="<HUGGING_FACE_TOKEN>"
terraform apply
```

### GCP
```bash
cd terraform-gcp
terraform init
export TF_VAR_project_id="<GCP_PROJECT_ID>"
export TF_VAR_hf_token="<HUGGING_FACE_TOKEN>"
terraform apply
```

Sau khi apply xong, đợi thêm vài phút để container tải model xong rồi mới gọi API.

## 7. Cách kiểm tra API
Dùng output của Terraform để lấy endpoint public, sau đó gọi thử:

```bash
curl -X POST http://<endpoint>/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/gemma-4-E2B-it",
    "messages": [
      {"role": "user", "content": "Hello"}
    ],
    "max_tokens": 64
  }'
```

Nếu nhận được JSON response từ model, nghĩa là hạ tầng và vLLM đã chạy đúng.

## 8. Dọn dẹp tài nguyên
Cả AWS và GCP đều có tài nguyên tốn phí, đặc biệt là GPU, NAT và Load Balancer. Sau khi kiểm tra xong, phải xóa hạ tầng ngay:

```bash
terraform destroy
```

## 9. Mục tiêu học được từ dự án
Sau khi hoàn thành lab này, người học sẽ hiểu:
- cách mô tả hạ tầng cloud bằng Terraform
- cách dựng private GPU inference server
- cách dùng NAT và Load Balancer để kết nối private workload ra ngoài
- cách chạy model gated của Hugging Face bằng token
- cách mở rộng cùng một bài lab sang AWS và GCP
