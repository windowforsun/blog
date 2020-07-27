깃 명령어, git 명령어, git command


로컬 브랜치 만들기$ git checkout -b <branch-name>

로컬 브랜치 원격 올리기$ git push <remote-name> <branch-name>

원격 브랜치 로컬로 가져오기$ git checkout -t <remote-name>/<branch-name>

원격 브랜치 로컬 브랜치에서 트래킹(연동)$ git branch --set-upstream-to <remote-name>/<branch-name>

로컬 브랜치 삭제하기$ git branch -d <branch-name>, git branch -D <branch-name>

원격 브랜치 삭제하기$ 로컬 브랜치 삭제 후 git push <remote-name> :<branch-name>, git push <remote-name> -d <branch-name>

원격 브랜치 리스트$ git branch -r

로컬 브랜치 리스트$ git branchㅣ

모든 브랜치 리스트$ git branch -a

로컬에서 원격 브랜치 리스트 갱신$ git remote update, git fetch

특정 커밋 시점으로 되돌리기$ git reset --soft <commit-hash>, git reset --hard <commit-hash>

로그 조회 트리$ git log --graph

로그 조회 한줄로$ git log --graph --oneline

로그 조회 파일 내용$ git log --name-only

로그 조회 최근 n개 커밋$ git log -<n>

특정 commit 파일 변경 내용$ git show <commit-hash>

track file to untrack$ git checkout <filename>, git rm --cache <filename>

stage to unstate$ git reset HEAD <filename>, git reset HEAD 

브랜치 이름 변경, 수정$ git branch -m <before-name> <new-name>

리모트 브랜치 삭제$ git push <remote-name> :<branch-name>

로컬 브랜치 리모트 브랜치 트래킹$ git branch --set-upstream-to=<리모트이름>/<리모트브랜치이름> <로컬브랜치이름>

존재하는 브랜치 커밋확인$ git branch -v, 리모트까지 포함해서 확인$ git branch -va

변경 파일 전체 수정 이전 커밋으로 되돌리기$ git reset --hard

변경한 특정 파일 수정 이전 커밋으로 되돌리기$ git checkout -- <파일경로 이름>
