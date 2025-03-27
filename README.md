# Projeto-FTT

from fastapi import FastAPI, HTTPException, Depends
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey, DateTime
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, relationship, Session
import datetime
import os

# Configuração do banco de dados
DATABASE_URL = os.getenv("DATABASE_URL", "postgresql+psycopg2://user:password@localhost/unievangelica_db")
engine = create_engine(DATABASE_URL, connect_args={"sslmode": "disable"})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# Inicialização do FastAPI
app = FastAPI()

# Modelos do banco de dados
class Bloco(Base):
    __tablename__ = "blocos"
    id = Column(Integer, primary_key=True, index=True)
    nome = Column(String, unique=True, index=True)
    salas = relationship("Sala", back_populates="bloco")

class Sala(Base):
    __tablename__ = "salas"
    id = Column(Integer, primary_key=True, index=True)
    numero = Column(String, index=True)
    capacidade = Column(Integer)
    recursos = Column(String)
    bloco_id = Column(Integer, ForeignKey("blocos.id"))
    bloco = relationship("Bloco", back_populates="salas")
    reservas = relationship("Reserva", back_populates="sala")

class Reserva(Base):
    __tablename__ = "reservas"
    id = Column(Integer, primary_key=True, index=True)
    sala_id = Column(Integer, ForeignKey("salas.id"))
    data_inicio = Column(DateTime)
    data_fim = Column(DateTime)
    coordenador = Column(String)
    motivo = Column(String)
    sala = relationship("Sala", back_populates="reservas")

# Criar tabelas no banco de dados
Base.metadata.create_all(bind=engine)

# Dependência do banco de dados
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Endpoints da API
@app.post("/blocos/")
def criar_bloco(nome: str, db: Session = Depends(get_db)):
    bloco = Bloco(nome=nome)
    db.add(bloco)
    db.commit()
    db.refresh(bloco)
    return bloco

@app.post("/salas/")
def criar_sala(numero: str, capacidade: int, recursos: str, bloco_id: int, db: Session = Depends(get_db)):
    sala = Sala(numero=numero, capacidade=capacidade, recursos=recursos, bloco_id=bloco_id)
    db.add(sala)
    db.commit()
    db.refresh(sala)
    return sala

@app.post("/reservas/")
def criar_reserva(sala_id: int, data_inicio: datetime.datetime, data_fim: datetime.datetime, coordenador: str, motivo: str, db: Session = Depends(get_db)):
    conflito = db.query(Reserva).filter(
        Reserva.sala_id == sala_id,
        Reserva.data_inicio < data_fim,
        Reserva.data_fim > data_inicio
    ).first()
    if conflito:
        raise HTTPException(status_code=400, detail="Conflito de agendamento!")
    reserva = Reserva(sala_id=sala_id, data_inicio=data_inicio, data_fim=data_fim, coordenador=coordenador, motivo=motivo)
    db.add(reserva)
    db.commit()
    db.refresh(reserva)
    return reserva

# Executar o servidor
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="127.0.0.1", port=8000, reload=True)


